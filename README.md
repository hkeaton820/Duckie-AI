# Duckie-AI
An AI chatbot using the rubber duck theory to help build and finish ideas
import { useState, useEffect, useRef } from "react";
import { base44 } from "@/api/base44Client";
import { Menu, X, FolderOpen } from "lucide-react";
import { motion, AnimatePresence } from "framer-motion";
import Sidebar from "@/components/layout/Sidebar";
import MessageBubble from "@/components/chat/MessageBubble";
import TypingIndicator from "@/components/chat/TypingIndicator";
import ChatInput from "@/components/chat/ChatInput";
import EmptyState from "@/components/chat/EmptyState";
import NewProjectModal from "@/components/projects/NewProjectModal";

export default function Chat() {
  const [projects, setProjects] = useState([]);
  const [selectedProject, setSelectedProject] = useState(null);
  const [messages, setMessages] = useState([]);
  const [isTyping, setIsTyping] = useState(false);
  const [showNewProject, setShowNewProject] = useState(false);
  const [sidebarOpen, setSidebarOpen] = useState(false);
  const [conversationId] = useState(() => `conv_${Date.now()}`);
  const bottomRef = useRef(null);

  useEffect(() => {
    loadProjects();
  }, []);

  useEffect(() => {
    if (selectedProject) loadMessages(selectedProject.id);
    else setMessages([]);
  }, [selectedProject]);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, isTyping]);

  const loadProjects = async () => {
    const data = await base44.entities.Project.list("-created_date");
    setProjects(data);
  };

  const loadMessages = async (projectId) => {
    const data = await base44.entities.Message.filter({ project_id: projectId }, "created_date", 100);
    setMessages(data);
  };

  const handleSend = async (content) => {
    const projectId = selectedProject?.id || "general";

    // Save user message
    const userMsg = await base44.entities.Message.create({
      project_id: projectId,
      role: "user",
      content,
      conversation_id: conversationId,
    });
    setMessages((prev) => [...prev, userMsg]);
    setIsTyping(true);

    // Build context for AI
    const context = selectedProject
      ? `You are an AI assistant helping with the project: "${selectedProject.title}". ${selectedProject.description ? `Project goal: ${selectedProject.description}.` : ""} Help the user brainstorm ideas, plan steps, solve problems, and turn their vision into reality. Be practical, encouraging, and creative.`
      : `You are Duckie, an AI assistant inspired by the rubber duck debugging theory — where explaining your problem out loud (even to a rubber duck) helps you find the solution. Help users clarify their thoughts, brainstorm ideas, and work through their projects by asking good questions and thinking alongside them. Be practical, creative, and encouraging.`;

    const history = messages.slice(-10).map((m) => `${m.role === "user" ? "User" : "Assistant"}: ${m.content}`).join("\n");
    const prompt = `${context}\n\nConversation so far:\n${history}\n\nUser: ${content}\n\nAssistant:`;

    try {
      const response = await base44.integrations.Core.InvokeLLM({ prompt });

      const aiMsg = await base44.entities.Message.create({
        project_id: projectId,
        role: "assistant",
        content: response,
        conversation_id: conversationId,
      });
      setMessages((prev) => [...prev, aiMsg]);
    } catch (err) {
      const isLimitError = err?.message?.toLowerCase().includes("limit");
      setMessages((prev) => [
        ...prev,
        {
          id: `err_${Date.now()}`,
          role: "assistant",
          content: isLimitError
            ? "🚫 Quack! I've hit my monthly AI usage limit. The app owner needs to upgrade their plan to keep chatting."
            : "😔 Something went wrong. Please try again in a moment.",
        },
      ]);
    } finally {
      setIsTyping(false);
    }
  };

  const handleNewProject = async (data) => {
    const project = await base44.entities.Project.create(data);
    setProjects((prev) => [project, ...prev]);
    setSelectedProject(project);
    setShowNewProject(false);
  };

  const handleDeleteProject = async (id) => {
    await base44.entities.Project.delete(id);
    setProjects((prev) => prev.filter((p) => p.id !== id));
    if (selectedProject?.id === id) setSelectedProject(null);
  };

  const handleNewChat = () => {
    setSelectedProject(null);
    setMessages([]);
  };

  return (
    <div className="flex h-screen bg-background overflow-hidden">
      {/* Mobile overlay */}
      <AnimatePresence>
        {sidebarOpen && (
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            className="fixed inset-0 bg-black/50 z-20 lg:hidden"
            onClick={() => setSidebarOpen(false)}
          />
        )}
      </AnimatePresence>

      {/* Sidebar — desktop always visible, mobile slide-in */}
      <div className={`
        fixed lg:relative inset-y-0 left-0 z-30 w-64 transition-transform duration-300 lg:translate-x-0
        ${sidebarOpen ? "translate-x-0" : "-translate-x-full"}
      `}>
        <Sidebar
          projects={projects}
          selectedProject={selectedProject}
          onSelectProject={(p) => { setSelectedProject(p); setSidebarOpen(false); }}
          onNewProject={() => setShowNewProject(true)}
          onDeleteProject={handleDeleteProject}
          onNewChat={handleNewChat}
        />
      </div>

      {/* Main chat area */}
      <div className="flex-1 flex flex-col min-w-0 bg-[#FFF8E1]">
        {/* Header */}
        <div className="flex items-center gap-3 px-5 py-3.5 border-b border-[#F5C842] bg-[#F5C842]/30 backdrop-blur-sm">
          <button
            className="lg:hidden text-[#7A5C1E] hover:text-[#3D2B00] transition-colors"
            onClick={() => setSidebarOpen(true)}
          >
            <Menu className="w-5 h-5" />
          </button>

          <div className="flex items-center gap-2 flex-1 min-w-0">
            {selectedProject ? (
              <>
                <span className="text-xl">{selectedProject.emoji || "📁"}</span>
                <div className="min-w-0">
                  <h1 className="font-black text-[#3D2B00] text-sm truncate">{selectedProject.title}</h1>
                  {selectedProject.description && (
                    <p className="text-xs text-[#7A5C1E] truncate">{selectedProject.description}</p>
                  )}
                </div>
              </>
            ) : (
              <div className="flex items-center gap-2">
                <span className="text-base">✨</span>
                <h1 className="font-black text-[#3D2B00] text-sm">General Chat</h1>
              </div>
            )}
          </div>

          {selectedProject && (
            <span className="text-xs text-[#7A5C1E] hidden sm:block font-medium">
              {messages.length} messages
            </span>
          )}
        </div>

        {/* Messages */}
        <div className="flex-1 overflow-y-auto px-4 py-6 space-y-4">
          {messages.length === 0 && !isTyping ? (
            <EmptyState
              projectTitle={selectedProject?.title}
              onSuggestion={handleSend}
            />
          ) : (
            <>
              {messages.map((msg) => (
                <MessageBubble key={msg.id} message={msg} />
              ))}
              {isTyping && <TypingIndicator />}
              <div ref={bottomRef} />
            </>
          )}
        </div>

        {/* Input */}
        <div className="px-4 pb-4 pt-2">
          <ChatInput
            onSend={handleSend}
            disabled={isTyping}
            placeholder={
              selectedProject
                ? `Ask about "${selectedProject.title}"...`
                : "Describe your idea or ask anything..."
            }
          />
          <p className="text-center text-xs text-[#A07820]/70 mt-2 font-medium font-nunito">
            Powered by AI • AI can make mistakes. Verify important information.
          </p>
        </div>
      </div>

      <NewProjectModal
        open={showNewProject}
        onClose={() => setShowNewProject(false)}
        onCreate={handleNewProject}
      />
    </div>
  );
}
