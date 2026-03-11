import React, { useState, useEffect, useRef } from 'react';
import { Send, Paperclip, User, Bot, Check, CheckCheck, MoreVertical, Phone, Video, Search, Image as ImageIcon, FileText, X } from 'lucide-react';
import { motion, AnimatePresence } from 'motion/react';
import { Message, ClientData } from './types';
import { processChatMessage } from './services/gemini';

const INITIAL_CLIENT_DATA: ClientData = {
  metadata: {
    transaction_id: 'SA-AUTO-GEN',
    status: 'active',
    stage: 'Intake',
    urgency: 'low',
    missing_docs: [],
    last_milestone: null,
    next_action: null,
    last_admin_update: null,
    client_satisfaction_logic: 'neutral',
    document_vault: [],
    document_detected: null,
  },
  client: {
    name: null,
    id: null,
    phone: null,
    email: null,
    verification: {
      id_provided: false,
      id_valid: false,
    },
  },
  transaction: {
    type: null,
    address: null,
    price: null,
    mortgage: null,
  },
  files: {
    id_copy: false,
    tabu_extract: false,
    other: [],
  },
  reminders_sent: [],
  internal_status: 'intake_active',
};

const parseMessageContent = (content: string) => {
  if (content.includes('[MESSAGE]')) {
    const messagePart = content.split('[DATA]')[0].replace('[MESSAGE]', '').trim();
    return messagePart;
  }
  return content;
};

export default function App() {
  const [messages, setMessages] = useState<Message[]>([
    {
      id: '1',
      role: 'assistant',
      content: '[MESSAGE] שלום רב, אני Case Manager בכיר במשרד עורכי הדין סהר אלון. אני כאן כדי ללוות אותך לאורך כל עסקת המקרקעין שלך בצורה המקצועית ביותר. באיזו עסקה מדובר? (מכירה, קנייה או שכירות) [DATA] {"metadata": {"stage": "Intake", "urgency": "low", "client_satisfaction_logic": "neutral", "document_detected": null, "missing_docs": ["סוג עסקה", "שם מלא", "תעודת זהות", "כתובת נכס", "מחיר", "משכנתה"]}, "client": {"name": null, "id": null, "verification": {"id_provided": false}}, "transaction": {"type": null, "address": null, "price": null, "mortgage": null}}',
      timestamp: new Date(),
    },
  ]);
  const [input, setInput] = useState('');
  const [clientData, setClientData] = useState<ClientData>(INITIAL_CLIENT_DATA);
  const [isTyping, setIsTyping] = useState(false);
  const [showInternal, setShowInternal] = useState(false);
  const [attachments, setAttachments] = useState<{ name: string; type: string; data: string }[]>([]);
  
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const fileInputRef = useRef<HTMLInputElement>(null);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  useEffect(() => {
    scrollToBottom();
  }, [messages, isTyping]);

  // Reminder Logic
  useEffect(() => {
    const checkReminders = () => {
      if (clientData.internal_status !== 'intake_active') return;
      
      const lastMsg = messages[messages.length - 1];
      if (!lastMsg || lastMsg.role !== 'assistant') return;

      const now = new Date().getTime();
      const lastTime = new Date(lastMsg.timestamp).getTime();
      const hoursPassed = (now - lastTime) / (1000 * 60 * 60);

      const triggerReminder = (hours: number, text: string) => {
        if (!clientData.reminders_sent.includes(hours)) {
          const reminderMsg: Message = {
            id: `reminder-${hours}-${Date.now()}`,
            role: 'assistant',
            content: text,
            timestamp: new Date(),
          };
          setMessages(prev => [...prev, reminderMsg]);
          setClientData(prev => ({
            ...prev,
            reminders_sent: [...prev.reminders_sent, hours]
          }));
        }
      };

      if (hoursPassed >= 28) {
        triggerReminder(28, 'שלום, אני רואה שעדיין לא התקבלו כל הפרטים. חשוב לנו להתקדם בעסקה כדי לשמור על הזכויות שלך. האם יש משהו שמעכב אותך?');
      } else if (hoursPassed >= 24) {
        triggerReminder(24, 'היי, רק תזכורת קטנה לגבי הפרטים שביקשתי. אני כאן לכל שאלה אם משהו לא ברור.');
      }
    };

    const interval = setInterval(checkReminders, 10000); // Check every 10 seconds
    return () => clearInterval(interval);
  }, [messages, clientData]);

  const handleSend = async () => {
    if (!input.trim() && attachments.length === 0) return;

    const userMessage: Message = {
      id: Date.now().toString(),
      role: 'user',
      content: input,
      timestamp: new Date(),
      attachments: attachments.length > 0 ? [...attachments] : undefined,
    };

    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setAttachments([]);
    setIsTyping(true);

    try {
      const { updatedData, response } = await processChatMessage([...messages, userMessage], clientData);
      
      setClientData(updatedData);
      
      const assistantMessage: Message = {
        id: (Date.now() + 1).toString(),
        role: 'assistant',
        content: response,
        timestamp: new Date(),
      };
      
      setMessages(prev => [...prev, assistantMessage]);
    } catch (error) {
      console.error('Error processing message:', error);
      const errorMessage: Message = {
        id: (Date.now() + 1).toString(),
        role: 'assistant',
        content: 'מצטער, חלה שגיאה בעיבוד ההודעה. אנא נסה שוב.',
        timestamp: new Date(),
      };
      setMessages(prev => [...prev, errorMessage]);
    } finally {
      setIsTyping(false);
    }
  };

  const handleFileUpload = (e: React.ChangeEvent<HTMLInputElement>) => {
    const files = e.target.files;
    if (!files) return;

    Array.from(files).forEach((file: File) => {
      const reader = new FileReader();
      reader.onloadend = () => {
        const base64 = reader.result as string;
        const data = base64.split(',')[1];
        setAttachments(prev => [...prev, {
          name: file.name,
          type: file.type,
          data: data
        }]);
      };
      reader.readAsDataURL(file);
    });
  };

  return (
    <div className="flex h-screen bg-[#f0f2f5] font-sans text-right" dir="rtl">
      {/* Main Chat Area */}
      <div className="flex flex-col flex-1 max-w-4xl mx-auto bg-white shadow-2xl overflow-hidden relative">
        
        {/* Chat Header */}
        <header className="bg-[#00a884] text-white p-3 flex items-center justify-between shadow-md z-10">
          <div className="flex items-center gap-3">
            <div className="w-10 h-10 rounded-full bg-white/20 flex items-center justify-center overflow-hidden">
              <img 
                src="https://picsum.photos/seed/lawyer/100/100" 
                alt="Sahar Alon" 
                className="w-full h-full object-cover"
                referrerPolicy="no-referrer"
              />
            </div>
            <div>
              <div className="flex items-center gap-2">
                <h1 className="font-bold text-lg leading-tight">עו"ד סהר אלון - משרד עו"ד</h1>
                {clientData.status && (
                  <span className={`text-[10px] px-1.5 py-0.5 rounded-full font-bold border ${
                    clientData.status === 'completed' 
                      ? 'bg-green-500/20 border-green-400 text-green-100' 
                      : clientData.status === 'message_received'
                      ? 'bg-blue-500/20 border-blue-400 text-blue-100'
                      : 'bg-yellow-500/20 border-yellow-400 text-yellow-100'
                  }`}>
                    {clientData.status === 'completed' ? 'הושלם' : 
                     clientData.status === 'message_received' ? 'בטיפול' : 'ממתין לפרטים'}
                  </span>
                )}
              </div>
              <p className="text-xs text-white/80">מחובר/ת</p>
            </div>
          </div>
          <div className="flex items-center gap-5">
            <Video size={20} className="cursor-pointer opacity-80 hover:opacity-100" />
            <Phone size={20} className="cursor-pointer opacity-80 hover:opacity-100" />
            <div className="w-[1px] h-6 bg-white/20 mx-1"></div>
            <Search size={20} className="cursor-pointer opacity-80 hover:opacity-100" />
            <MoreVertical size={20} className="cursor-pointer opacity-80 hover:opacity-100" />
          </div>
        </header>

        {/* Messages Area */}
        <div 
          className="flex-1 overflow-y-auto p-4 space-y-3 bg-[#e5ddd5] relative"
          style={{ backgroundImage: 'url("https://user-images.githubusercontent.com/15075759/28719144-86dc0f70-73b1-11e7-911d-60d70fcded21.png")', backgroundBlendMode: 'overlay' }}
        >
          <div className="flex justify-center mb-4">
            <span className="bg-[#d9fdd3] text-[#54656f] text-[11px] px-3 py-1 rounded-lg shadow-sm uppercase font-medium">היום</span>
          </div>

          <AnimatePresence initial={false}>
            {messages.map((msg) => (
              <motion.div
                key={msg.id}
                initial={{ opacity: 0, y: 10, scale: 0.95 }}
                animate={{ opacity: 1, y: 0, scale: 1 }}
                className={`flex ${msg.role === 'user' ? 'justify-start' : 'justify-end'}`}
              >
                <div 
                  className={`max-w-[85%] p-2 rounded-lg shadow-sm relative group ${
                    msg.role === 'user' 
                      ? 'bg-[#d9fdd3] rounded-tr-none' 
                      : 'bg-white rounded-tl-none'
                  }`}
                >
                  {msg.attachments && msg.attachments.length > 0 && (
                    <div className="mb-2 space-y-1">
                      {msg.attachments.map((att, idx) => (
                        <div key={idx} className="bg-black/5 p-2 rounded flex items-center gap-2">
                          {att.type.startsWith('image/') ? <ImageIcon size={16} /> : <FileText size={16} />}
                          <span className="text-xs truncate max-w-[150px]">{att.name}</span>
                        </div>
                      ))}
                    </div>
                  )}
                  <p className="text-[14.2px] text-[#111b21] whitespace-pre-wrap leading-relaxed">
                    {parseMessageContent(msg.content)}
                  </p>
                  <div className="flex items-center justify-end gap-1 mt-1">
                    <span className="text-[10px] text-[#667781]">
                      {msg.timestamp.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
                    </span>
                    {msg.role === 'user' && <CheckCheck size={14} className="text-[#53bdeb]" />}
                  </div>
                </div>
              </motion.div>
            ))}
          </AnimatePresence>

          {isTyping && (
            <motion.div 
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              className="flex justify-end"
            >
              <div className="bg-white p-3 rounded-lg rounded-tl-none shadow-sm flex gap-1">
                <span className="w-1.5 h-1.5 bg-gray-400 rounded-full animate-bounce"></span>
                <span className="w-1.5 h-1.5 bg-gray-400 rounded-full animate-bounce [animation-delay:0.2s]"></span>
                <span className="w-1.5 h-1.5 bg-gray-400 rounded-full animate-bounce [animation-delay:0.4s]"></span>
              </div>
            </motion.div>
          )}
          <div ref={messagesEndRef} />
        </div>

        {/* Input Area */}
        <footer className="bg-[#f0f2f5] p-2 flex items-end gap-2 border-t border-gray-200">
          <button 
            onClick={() => fileInputRef.current?.click()}
            className="p-2 text-[#54656f] hover:bg-gray-200 rounded-full transition-colors"
          >
            <Paperclip size={24} />
          </button>
          <input 
            type="file" 
            ref={fileInputRef} 
            onChange={handleFileUpload} 
            className="hidden" 
            multiple
          />
          
          <div className="flex-1 bg-white rounded-lg p-1 shadow-sm flex flex-col">
            {attachments.length > 0 && (
              <div className="flex flex-wrap gap-2 p-2 border-b border-gray-100">
                {attachments.map((att, idx) => (
                  <div key={idx} className="bg-[#f0f2f5] px-2 py-1 rounded-full flex items-center gap-1 text-xs">
                    <span className="truncate max-w-[100px]">{att.name}</span>
                    <button onClick={() => setAttachments(prev => prev.filter((_, i) => i !== idx))}>
                      <X size={14} className="text-gray-500 hover:text-red-500" />
                    </button>
                  </div>
                ))}
              </div>
            )}
            <textarea
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === 'Enter' && !e.shiftKey) {
                  e.preventDefault();
                  handleSend();
                }
              }}
              placeholder="הקלד/י הודעה"
              className="w-full p-2 bg-transparent focus:outline-none resize-none max-h-32 min-h-[40px] text-[15px]"
              rows={1}
            />
          </div>
          
          <button 
            onClick={handleSend}
            disabled={!input.trim() && attachments.length === 0}
            className={`p-3 rounded-full transition-all ${
              !input.trim() && attachments.length === 0 
                ? 'text-gray-400' 
                : 'bg-[#00a884] text-white hover:bg-[#008f70]'
            }`}
          >
            <Send size={20} className={input.trim() || attachments.length > 0 ? 'translate-x-[-2px]' : ''} />
          </button>
        </footer>

        {/* Internal View Toggle */}
        <button 
          onClick={() => setShowInternal(!showInternal)}
          className="absolute top-20 left-4 bg-black/80 text-white text-[10px] px-2 py-1 rounded-md z-20 hover:bg-black"
        >
          {showInternal ? 'הסתר תצוגה פנימית' : 'הצג תצוגה פנימית'}
        </button>
      </div>

      {/* Internal Data Panel (Sidebar) */}
      <AnimatePresence>
        {showInternal && (
          <motion.div
            initial={{ x: '100%' }}
            animate={{ x: 0 }}
            exit={{ x: '100%' }}
            className="w-80 bg-white border-r border-gray-200 shadow-xl overflow-y-auto p-4 z-30"
          >
            <div className="flex items-center justify-between mb-6">
              <h2 className="font-bold text-lg text-gray-800">נתוני לקוח (פנימי)</h2>
              <button onClick={() => setShowInternal(false)} className="text-gray-400 hover:text-gray-600">
                <X size={20} />
              </button>
            </div>

            <div className="space-y-6">
              <section>
                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">סימולציה</h3>
                <button 
                  onClick={() => {
                    const lastMsg = [...messages].reverse().find(m => m.role === 'assistant');
                    if (lastMsg) {
                      lastMsg.timestamp = new Date(Date.now() - (25 * 60 * 60 * 1000)); // 25 hours ago
                      setMessages([...messages]);
                    }
                  }}
                  className="w-full py-2 bg-orange-500 text-white text-xs rounded hover:bg-orange-600 transition-colors"
                >
                  סמלץ מעבר של 25 שעות
                </button>
              </section>

              <section>
                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">JSON סטטוס</h3>
                <pre className="bg-gray-900 text-green-400 p-3 rounded-lg text-[10px] overflow-x-auto font-mono" dir="ltr">
                  {JSON.stringify(clientData, null, 2)}
                </pre>
              </section>

              <section>
                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">סטטוס תיק</h3>
                <div className="space-y-2 text-xs">
                  <div className="bg-gray-800 p-2 rounded">
                    <span className="text-gray-400 block">מזהה עסקה:</span>
                    <span className="text-white font-mono">{clientData.metadata.transaction_id}</span>
                  </div>
                  <div className="bg-gray-800 p-2 rounded">
                    <span className="text-gray-400 block">שלב נוכחי:</span>
                    <span className="text-white font-bold">
                      {clientData.metadata.stage === 'Intake' ? 'קליטה' :
                       clientData.metadata.stage === 'Drafts' ? 'טיוטות' :
                       clientData.metadata.stage === 'Signing' ? 'חתימה' :
                       clientData.metadata.stage === 'Tax' ? 'דיווח מס' :
                       clientData.metadata.stage === 'Warning' ? 'הערת אזהרה' :
                       clientData.metadata.stage === 'Final' ? 'סיום ורישום' : clientData.metadata.stage}
                    </span>
                  </div>
                  <div className="bg-gray-800 p-2 rounded">
                    <span className="text-gray-400 block">שלב אחרון:</span>
                    <span className="text-white">{clientData.metadata.last_milestone || '-'}</span>
                  </div>
                  <div className="bg-gray-800 p-2 rounded">
                    <span className="text-gray-400 block">שלב הבא:</span>
                    <span className="text-white">{clientData.metadata.next_action || '-'}</span>
                  </div>
                  <div className="bg-gray-800 p-2 rounded">
                    <span className="text-gray-400 block">מסמך שזוהה:</span>
                    <span className="text-white">{clientData.metadata.document_detected || '-'}</span>
                  </div>
                  <div className="bg-gray-800 p-2 rounded">
                    <span className="text-gray-400 block">שביעות רצון:</span>
                    <span className={`font-bold ${
                      clientData.metadata.client_satisfaction_logic === 'happy' ? 'text-green-400' :
                      clientData.metadata.client_satisfaction_logic === 'urgent' ? 'text-red-400' : 'text-blue-400'
                    }`}>
                      {clientData.metadata.client_satisfaction_logic === 'happy' ? 'חיובי' :
                       clientData.metadata.client_satisfaction_logic === 'urgent' ? 'דחוף/כועס' : 'ניטרלי'}
                    </span>
                  </div>
                </div>
              </section>

              <section>
                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">פרטי לקוח</h3>
                <div className="border border-gray-100 rounded-lg overflow-hidden mb-4">
                  <table className="w-full text-sm">
                    <tbody className="divide-y divide-gray-100">
                      {[
                        ['שם', clientData.client.name],
                        ['ת"ז', clientData.client.id],
                        ['טלפון', clientData.client.phone],
                        ['מייל', clientData.client.email],
                        ['אימות ת"ז', clientData.client.verification.id_valid ? 'תקין' : clientData.client.verification.id_provided ? 'בבדיקה' : 'טרם סופק'],
                      ].map(([label, value]) => (
                        <tr key={label}>
                          <td className="p-2 bg-gray-50 font-medium text-gray-600 w-1/3">{label}</td>
                          <td className="p-2 text-gray-800">{value || '-'}</td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>

                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">פרטי עסקה</h3>
                <div className="border border-gray-100 rounded-lg overflow-hidden">
                  <table className="w-full text-sm">
                    <tbody className="divide-y divide-gray-100">
                      {[
                        ['סוג', clientData.transaction.type === 'sale' ? 'מכירה' : clientData.transaction.type === 'purchase' ? 'קנייה' : clientData.transaction.type === 'rent' ? 'שכירות' : '-'],
                        ['כתובת', clientData.transaction.address],
                        ['מחיר', clientData.transaction.price],
                        ['משכנתה', clientData.transaction.mortgage === 'yes' ? 'כן' : clientData.transaction.mortgage === 'no' ? 'לא' : clientData.transaction.mortgage === 'pending' ? 'בבדיקה' : '-'],
                      ].map(([label, value]) => (
                        <tr key={label}>
                          <td className="p-2 bg-gray-50 font-medium text-gray-600 w-1/3">{label}</td>
                          <td className="p-2 text-gray-800">{value || '-'}</td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              </section>

              <section>
                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">ארכיון מסמכים (Vault)</h3>
                <div className="space-y-1">
                  {clientData.metadata.document_vault.length > 0 ? (
                    clientData.metadata.document_vault.map((file, i) => (
                      <div key={i} className="text-xs p-2 bg-blue-900/30 text-blue-300 rounded border border-blue-800 flex items-center gap-2">
                        <FileText size={12} />
                        {file}
                      </div>
                    ))
                  ) : (
                    <p className="text-xs text-gray-500 italic">הארכיון ריק</p>
                  )}
                </div>
              </section>

              <section>
                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">מסמכים</h3>
                <div className="space-y-2">
                  <div className="flex items-center justify-between p-2 bg-gray-50 rounded border border-gray-100 text-xs">
                    <span className="text-gray-600">צילום ת"ז:</span>
                    {clientData.files.id_copy ? <Check className="w-3 h-3 text-green-500" /> : <X className="w-3 h-3 text-red-500" />}
                  </div>
                  <div className="flex items-center justify-between p-2 bg-gray-50 rounded border border-gray-100 text-xs">
                    <span className="text-gray-600">נסח טאבו:</span>
                    {clientData.files.tabu_extract ? <Check className="w-3 h-3 text-green-500" /> : <X className="w-3 h-3 text-red-500" />}
                  </div>
                  {clientData.files.other.length > 0 && (
                    <div className="p-2 bg-gray-50 rounded border border-gray-100 text-xs">
                      <span className="text-gray-600 block mb-1">אחרים:</span>
                      <ul className="list-disc list-inside text-gray-800">
                        {clientData.files.other.map((f, i) => <li key={i}>{f}</li>)}
                      </ul>
                    </div>
                  )}
                </div>
              </section>

              <section>
                <h3 className="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">מידע חסר</h3>
                <div className="flex flex-wrap gap-1">
                  {clientData.metadata.missing_docs.length > 0 ? (
                    clientData.metadata.missing_docs.map((info, i) => (
                      <span key={i} className="text-[10px] px-2 py-0.5 bg-red-50 text-red-600 rounded-full border border-red-100">
                        {info}
                      </span>
                    ))
                  ) : (
                    <p className="text-xs text-gray-400 italic">אין מידע חסר</p>
                  )}
                </div>
              </section>
            </div>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
}
