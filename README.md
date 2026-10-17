# My-Jarvis-
My Jarvis 
?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.jarvis.ai">

    <!-- Permissions required for full phone voice operation -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE" />
    <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.CALL_PHONE" />
    <uses-permission android:name="android.permission.READ_CONTACTS" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="JARVIS AI"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@android:style/Theme.DeviceDefault.NoActionBar">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:configChanges="orientation|keyboardHidden|screenSize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Broadcast receiver for the exact 5:00 AM alarm -->
        <receiver android:name=".AlarmReceiver" android:exported="false" />

    </application>
</manifest>
import React, { useState, useEffect } from 'react';
import { AppSettings, ChatMessage, SupportedLanguage } from './types';
import { speechEngine } from './services/speechEngine';
import { wakeWordDetector } from './services/wakeWord';
import { androidBridge } from './services/androidBridge';
import { JarvisOrb } from './components/JarvisOrb';
import { AlarmModule } from './components/AlarmModule';
import { IndiaNews } from './components/IndiaNews';
import { VideoStudio } from './components/VideoStudio';
import { SocialStudio } from './components/SocialStudio';
import { AppBuilder } from './components/AppBuilder';
import { SettingsModal } from './components/SettingsModal';

const DEFAULT_SETTINGS: AppSettings = {
  language: 'hi-IN',
  assistantName: 'JARVIS',
  wakeWordEnabled: true,
  voiceSpeed: 1.0,
  voicePitch: 1.0,
  selectedVoiceURI: '',
  dailyAlarmEnabled: true,
  alarmTime: '05:00',
  alarmDays: [0, 1, 2, 3, 4, 5, 6],
  youtubeUploadEnabled: false,
  instagramPublishEnabled: false,
  facebookPublishEnabled: false,
  backendUrl: 'http://localhost:5000'
};

export const App: React.FC = () => {
  const [settings, setSettings] = useState<AppSettings>(() => {
    const saved = localStorage.getItem('jarvis_settings');
    return saved ? JSON.parse(saved) : DEFAULT_SETTINGS;
  });

  const [activeTab, setActiveTab] = useState<
    'talk' | 'chat' | 'controls' | 'alarm' | 'news' | 'video' | 'social' | 'builder'
  >('talk');

  const [assistantState, setAssistantState] = useState<'idle' | 'listening' | 'thinking' | 'speaking'>('idle');
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [inputText, setInputText] = useState('');
  const [showSettings, setShowSettings] = useState(false);

  useEffect(() => {
    localStorage.setItem('jarvis_settings', JSON.stringify(settings));
  }, [settings]);

  // Initial greeting upon startup
  useEffect(() => {
    const initialGreeting =
      settings.language === 'bho-IN'
        ? `Namaste! Hum ${settings.assistantName} baani. Batayi, hum raua khate ka kar sakatani?`
        : `Namaste! Main ${settings.assistantName} hoon. Bataiye, main aapki kya madad kar sakta hoon?`;

    setMessages([
      {
        id: '1',
        role: 'jarvis',
        text: initialGreeting,
        timestamp: new Date().toLocaleTimeString()
      }
    ]);
  }, []);

  // Background Wake Word listener cycle
  useEffect(() => {
    if (settings.wakeWordEnabled && assistantState === 'idle') {
      wakeWordDetector.start(settings.assistantName, () => {
        handleTriggerVoiceInput();
      });
    } else {
      wakeWordDetector.stop();
    }
    return () => wakeWordDetector.stop();
  }, [settings.wakeWordEnabled, settings.assistantName, assistantState]);

  // Voice command intent router
  const routeVoiceAction = (text: string): boolean => {
    const lower = text.toLowerCase();

    // 1. Language switching by voice
    if (lower.includes('hindi mein baat karo') || lower.includes('speak hindi')) {
      setSettings((s) => ({ ...s, language: 'hi-IN' }));
      respond('Ab se hum Hindi mein baat karenge.');
      return true;
    }
    if (lower.includes('bhojpuri mein') || lower.includes('speak bhojpuri')) {
      setSettings((s) => ({ ...s, language: 'bho-IN' }));
      respond('Theek ba, ab se hum Bhojpuri mein baat karab.');
      return true;
    }
    if (lower.includes('marathi mein') || lower.includes('speak marathi')) {
      setSettings((s) => ({ ...s, language: 'mr-IN' }));
      respond('नक्कीच, आता आपण मराठीमध्ये बोलूया.');
      return true;
    }
    if (lower.includes('speak english')) {
      setSettings((s) => ({ ...s, language: 'en-US' }));
      respond('Switched to English. How can I assist you?');
      return true;
    }

    // 2. Android App Intents
    if (lower.includes('youtube kholo') || lower.includes('open youtube')) {
      androidBridge.launchApp('youtube');
      respond('Opening YouTube.');
      return true;
    }
    if (lower.includes('whatsapp kholo') || lower.includes('open whatsapp')) {
      androidBridge.launchApp('whatsapp');
      respond('Opening WhatsApp.');
      return true;
    }
    if (lower.includes('camera kholo') || lower.includes('open camera')) {
      androidBridge.launchApp('camera');
      respond('Launching Camera.');
      return true;
    }
    if (lower.includes('flashlight on') || lower.includes('torch on')) {
      androidBridge.toggleFlashlight();
      respond('Flashlight toggled.');
      return true;
    }
    if (lower.includes('bluetooth settings')) {
      androidBridge.launchApp('bluetooth');
      respond('Opening Bluetooth settings.');
      return true;
    }

    // 3. YouTube Music Intents
    if (lower.includes('song chalao') || lower.includes('play')) {
      const query = text.replace(/song chalao|play|play karo|chalao/gi, '').trim();
      androidBridge.searchYouTube(query);
      respond(`Searching and playing ${query} on YouTube.`);
      return true;
    }

    // 4. Calling Intents
    if (lower.includes('ko call lagao') || lower.includes('ko phone karo')) {
      const contact = text.replace(/ko call lagao|ko phone karo|jarvis|hey/gi, '').trim();
      respond(`Calling ${contact} via Android dialer.`);
      androidBridge.initiatePhoneCall(contact);
      return true;
    }

    // 5. WhatsApp Message Drafting Intent
    if (lower.includes('whatsapp message bhejo') || lower.includes('message likho')) {
      const messageBody = text.replace(/whatsapp message bhejo|message likho|jarvis/gi, '').trim();
      androidBridge.draftWhatsAppMessage('', messageBody || 'Namaste!');
      respond('Drafting WhatsApp message for preview and confirmation.');
      return true;
    }

    return false;
  };

  const respond = (text: string) => {
    setMessages((prev) => [
      ...prev,
      { id: Date.now().toString(), role: 'jarvis', text, timestamp: new Date().toLocaleTimeString() }
    ]);
    setAssistantState('speaking');
    speechEngine.speak(
      text,
      settings.language,
      settings.selectedVoiceURI,
      settings.voiceSpeed,
      settings.voicePitch,
      () => setAssistantState('speaking'),
      () => setAssistantState('idle')
    );
  };

  const processUserQuery = async (queryText: string) => {
    setMessages((prev) => [
      ...prev,
      { id: Date.now().toString(), role: 'user', text: queryText, timestamp: new Date().toLocaleTimeString() }
    ]);

    // Check if intercepted by direct Android intent
    if (routeVoiceAction(queryText)) {
      return;
    }

    setAssistantState('thinking');
    try {
      const res = await fetch(`${settings.backendUrl}/api/chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: queryText,
          conversationHistory: messages.slice(-6),
          language: settings.language,
          assistantName: settings.assistantName
        })
      });

      const data = await res.json();
      if (data.reply) {
        respond(data.reply);
      } else {
        respond('I encountered an error understanding that query.');
      }
    } catch (e) {
      respond('Unable to reach AI backend server. Please verify your connection.');
      setAssistantState('idle');
    }
  };

  const handleTriggerVoiceInput = () => {
    if (assistantState === 'speaking') {
      speechEngine.stopSpeaking();
      setAssistantState('idle');
      return;
    }

    if (assistantState === 'listening') {
      speechEngine.stopListening();
      setAssistantState('idle');
      return;
    }

    setAssistantState('listening');
    speechEngine.startListening(
      settings.language,
      (transcript) => {
        setAssistantState('idle');
        processUserQuery(transcript);
      },
      (err) => {
        console.error(err);
        setAssistantState('idle');
      },
      () => {
        if (assistantState === 'listening') setAssistantState('idle');
      }
    );
  };

  return (
    <div className="min-h-screen bg-slate-950 text-white flex flex-col font-sans pb-16">
      {/* Futuristic Header Bar */}
      <header className="px-5 py-4 border-b border-slate-800 flex items-center justify-between bg-slate-900/60 backdrop-blur-md sticky top-0 z-40">
        <div className="flex items-center space-x-2">
          <div className="w-3 h-3 rounded-full bg-cyan-400 animate-pulse" />
          <h1 className="text-base font-black tracking-wider text-cyan-400 font-mono">
            {settings.assistantName} AI
          </h1>
          <span className="text-[10px] px-2 py-0.5 rounded-full bg-cyan-950 text-cyan-300 font-mono border border-cyan-800">
            {settings.language.toUpperCase()}
          </span>
        </div>

        <button
          onClick={() => setShowSettings(true)}
          className="p-2 rounded-lg bg-slate-800 hover:bg-slate-700 text-slate-300 text-xs font-mono"
        >
          ⚙ Settings
        </button>
      </header>

      {/* Navigation Sub-bar */}
      <nav className="flex space-x-2 px-4 py-2.5 overflow-x-auto bg-slate-900 border-b border-slate-800/80 text-xs scrollbar-none">
        {[
          { id: 'talk', label: 'Voice' },
          { id: 'chat', label: 'Chat' },
          { id: 'alarm', label: 'Alarm (5 AM)' },
          { id: 'news', label: 'India News' },
          { id: 'video', label: 'Video Studio' },
          { id: 'social', label: 'YouTube/Meta' },
          { id: 'builder', label: 'App Builder' }
        ].map((tab) => (
          <button
            key={tab.id}
            onClick={() => setActiveTab(tab.id as any)}
            className={`px-3 py-1.5 rounded-lg whitespace-nowrap font-medium transition ${
              activeTab === tab.id
                ? 'bg-cyan-500 text-black font-semibold'
                : 'text-slate-400 hover:text-white bg-slate-800/50'
            }`}
          >
            {tab.label}
          </button>
        ))}
      </nav>

      {/* Main Dynamic View Area */}
      <main className="flex-1 max-w-lg w-full mx-auto p-4 flex flex-col">
        {activeTab === 'talk' && (
          <div className="flex-1 flex flex-col justify-between items-center py-6">
            <JarvisOrb
              state={assistantState}
              assistantName={settings.assistantName}
              onClick={handleTriggerVoiceInput}
            />

            {/* Live Subtitle Transcript */}
            <div className="w-full bg-slate-900/80 border border-slate-800 p-4 rounded-xl min-h-24 max-h-36 overflow-y-auto">
              <span className="text-[10px] text-cyan-400 font-mono uppercase block mb-1">Live Output</span>
              <p className="text-sm text-slate-200">
                {messages[messages.length - 1]?.text || 'Tap the glowing reactor or say "Hey ' + settings.assistantName + '"'}
              </p>
            </div>

            {/* Quick Action Commands */}
            <div className="w-full grid grid-cols-2 gap-2 mt-4 text-[11px] font-mono">
              <button
                onClick={() => processUserQuery('Bhojpuri song chalao')}
                className="p-2.5 rounded-lg bg-slate-900 border border-slate-800 text-slate-300 text-left hover:border-cyan-500"
              >
                🎵 "Bhojpuri song chalao"
              </button>
              <button
                onClick={() => processUserQuery('Flashlight on karo')}
                className="p-2.5 rounded-lg bg-slate-900 border border-slate-800 text-slate-300 text-left hover:border-cyan-500"
              >
                🔦 "Flashlight on karo"
              </button>
              <button
                onClick={() => processUserQuery('Mummy ko phone karo')}
                className="p-2.5 rounded-lg bg-slate-900 border border-slate-800 text-slate-300 text-left hover:border-cyan-500"
              >
                📞 "Mummy ko call lagao"
              </button>
              <button
                onClick={() => processUserQuery('YouTube kholo')}
                className="p-2.5 rounded-lg bg-slate-900 border border-slate-800 text-slate-300 text-left hover:border-cyan-500"
              >
                ▶ "YouTube kholo"
              </button>
            </div>
          </div>
        )}

        {activeTab === 'chat' && (
          <div className="flex-1 flex flex-col justify-between">
            <div className="space-y-3 overflow-y-auto max-h-[60vh] pr-1">
              {messages.map((m) => (
                <div
                  key={m.id}
                  className={`p-3 rounded-xl max-w-[85%] text-xs ${
                    m.role === 'user'
                      ? 'ml-auto bg-cyan-600 text-black font-medium'
                      : 'mr-auto bg-slate-900 border border-slate-800 text-slate-200'
                  }`}
                >
                  <p>{m.text}</p>
                  <span className="text-[9px] opacity-60 block mt-1">{m.timestamp}</span>
                </div>
              ))}
            </div>

            <div className="flex space-x-2 mt-4">
              <input
                type="text"
                placeholder="Type query or command..."
                value={inputText}
                onChange={(e) => setInputText(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === 'Enter' && inputText) {
                    processUserQuery(inputText);
                    setInputText('');
                  }
                }}
                className="flex-1 bg-slate-900 border border-slate-800 rounded-lg px-3 py-2 text-xs text-white focus:outline-none"
              />
              <button
                onClick={() => {
                  if (inputText) {
                    processUserQuery(inputText);
                    setInputText('');
                  }
                }}
                className="bg-cyan-500 text-black px-4 py-2 rounded-lg text-xs font-bold"
              >
                Send
              </button>
            </div>
          </div>
        )}

        {activeTab === 'alarm' && (
          <AlarmModule language={settings.language} assistantName={settings.assistantName} />
        )}

        {activeTab === 'news' && (
          <IndiaNews language={settings.language} backendUrl={settings.backendUrl} />
        )}

        {activeTab === 'video' && <VideoStudio backendUrl={settings.backendUrl} />}

        {activeTab === 'social' && (
          <SocialStudio settings={settings} backendUrl={settings.backendUrl} />
        )}

        {activeTab === 'builder' && (
          <AppBuilder language={settings.language} backendUrl={settings.backendUrl} />
        )}
      </main>

      {/* Settings Modal */}
      {showSettings && (
        <SettingsModal
          settings={settings}
          onUpdate={setSettings}
          onClose={() => setShowSettings(false)}
        />
      )}
    </div>
  );
};

export default App;
import React from 'react';
import { AppSettings, SupportedLanguage } from '../types';
import { speechEngine } from '../services/speechEngine';

interface SettingsProps {
  settings: AppSettings;
  onUpdate: (updated: AppSettings) => void;
  onClose: () => void;
}

export const SettingsModal: React.FC<SettingsProps> = ({ settings, onUpdate, onClose }) => {
  const voices = speechEngine.getAvailableVoices();

  return (
    <div className="fixed inset-0 bg-black/80 backdrop-blur-sm z-50 flex items-center justify-center p-4">
      <div className="bg-slate-900 border border-slate-700 w-full max-w-md rounded-2xl p-6 text-white max-h-[90vh] overflow-y-auto">
        <div className="flex items-center justify-between mb-4 border-b border-slate-800 pb-3">
          <h2 className="text-lg font-bold text-cyan-400">JARVIS System Configuration</h2>
          <button onClick={onClose} className="text-slate-400 hover:text-white text-lg">✕</button>
        </div>

        <div className="space-y-4 text-xs">
          {/* Assistant Name Setting */}
          <div>
            <label className="block text-slate-400 mb-1">Assistant Call Name (Wake Word Identifier)</label>
            <input
              type="text"
              value={settings.assistantName}
              onChange={(e) => onUpdate({ ...settings, assistantName: e.target.value.toUpperCase() })}
              className="w-full bg-slate-800 border border-slate-700 px-3 py-2 rounded-lg text-cyan-300 font-mono focus:outline-none"
            />
          </div>

          {/* Primary Language Selection */}
          <div>
            <label className="block text-slate-400 mb-1">Primary Conversation Language</label>
            <select
              value={settings.language}
              onChange={(e) => onUpdate({ ...settings, language: e.target.value as SupportedLanguage })}
              className="w-full bg-slate-800 border border-slate-700 px-3 py-2 rounded-lg text-white focus:outline-none"
            >
              <option value="hi-IN">Hindi (हिंदी)</option>
              <option value="bho-IN">Bhojpuri (भोजपुरी)</option>
              <option value="mr-IN">Marathi (मराठी)</option>
              <option value="en-US">English</option>
            </select>
          </div>

          {/* Wake Word Engine Activation */}
          <div className="flex items-center justify-between p-3 bg-slate-950 rounded-lg border border-slate-800">
            <div>
              <div className="font-semibold text-white">"Hey {settings.assistantName}" Wake Word</div>
              <div className="text-[10px] text-slate-400">Listens continuously when app is in foreground</div>
            </div>
            <input
              type="checkbox"
              checked={settings.wakeWordEnabled}
              onChange={(e) => onUpdate({ ...settings, wakeWordEnabled: e.target.checked })}
              className="w-4 h-4 accent-cyan-500"
            />
          </div>

          {/* Voice Engine Selector */}
          <div>
            <label className="block text-slate-400 mb-1">TTS Voice Profile ({voices.length} found)</label>
            <select
              value={settings.selectedVoiceURI}
              onChange={(e) => onUpdate({ ...settings, selectedVoiceURI: e.target.value })}
              className="w-full bg-slate-800 border border-slate-700 px-3 py-2 rounded-lg text-white focus:outline-none"
            >
              <option value="">Default System Engine</option>
              {voices.map((v) => (
                <option key={v.voiceURI} value={v.voiceURI}>
                  {v.name} ({v.lang})
                </option>
              ))}
            </select>
          </div>

          {/* Voice Speed */}
          <div>
            <div className="flex justify-between text-slate-400 mb-1">
              <span>Speaking Speed</span>
              <span className="font-mono text-cyan-400">{settings.voiceSpeed}x</span>
            </div>
            <input
              type="range"
              min="0.7"
              max="1.5"
              step="0.1"
              value={settings.voiceSpeed}
              onChange={(e) => onUpdate({ ...settings, voiceSpeed: parseFloat(e.target.value) })}
              className="w-full accent-cyan-500"
            />
          </div>

          {/* Social Publishing Authorization Toggles */}
          <div className="border-t border-slate-800 pt-3">
            <span className="text-[11px] font-mono text-slate-400 uppercase tracking-wider block mb-2">
              Authorized API Publishing Gates
            </span>
            <div className="space-y-2">
              <label className="flex items-center justify-between">
                <span>YouTube Data API Uploads</span>
                <input
                  type="checkbox"
                  checked={settings.youtubeUploadEnabled}
                  onChange={(e) => onUpdate({ ...settings, youtubeUploadEnabled: e.target.checked })}
                  className="accent-red-500"
                />
              </label>
              <label className="flex items-center justify-between">
                <span>Instagram Graph API Publishing</span>
                <input
                  type="checkbox"
                  checked={settings.instagramPublishEnabled}
                  onChange={(e) => onUpdate({ ...settings, instagramPublishEnabled: e.target.checked })}
                  className="accent-pink-500"
                />
              </label>
              <label className="flex items-center justify-between">
                <span>Facebook Pages API Publishing</span>
                <input
                  type="checkbox"
                  checked={settings.facebookPublishEnabled}
                  onChange={(e) => onUpdate({ ...settings, facebookPublishEnabled: e.target.checked })}
                  className="accent-blue-500"
                />
              </label>
            </div>
          </div>

          {/* Backend Proxy Server URL */}
          <div>
            <label className="block text-slate-400 mb-1">Secure Backend URL</label>
            <input
              type="text"
              value={settings.backendUrl}
              onChange={(e) => onUpdate({ ...settings, backendUrl: e.target.value })}
              className="w-full bg-slate-800 border border-slate-700 px-3 py-2 rounded-lg text-slate-300 font-mono focus:outline-none"
            />
          </div>
        </div>

        <button
          onClick={onClose}
          className="w-full mt-6 bg-cyan-600 hover:bg-cyan-500 text-black font-bold py-2.5 rounded-xl transition"
        >
          Save & Return
        </button>
      </div>
    </div>
  );
};
import React, { useState } from 'react';
import { SupportedLanguage } from '../types';

export const AppBuilder: React.FC<{ language: SupportedLanguage; backendUrl: string }> = ({
  language,
  backendUrl
}) => {
  const [appDescription, setAppDescription] = useState('');
  const [isBuilding, setIsBuilding] = useState(false);
  const [generatedCode, setGeneratedCode] = useState<string | null>(null);

  const handleBuildPlan = async () => {
    if (!appDescription) return;
    setIsBuilding(true);
    try {
      const res = await fetch(`${backendUrl}/api/chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: `You are an expert Android Mobile App Developer. The user requested: "${appDescription}".
Generate:
1. Complete App Architecture & Feature list.
2. Complete functional React + Tailwind UI code.
3. Step-by-step Android Studio APK compilation command instructions using Capacitor/Gradle.`,
          language
        })
      });
      const data = await res.json();
      setGeneratedCode(data.reply);
    } catch (e) {
      setGeneratedCode('Error contacting AI App Builder Engine.');
    } finally {
      setIsBuilding(false);
    }
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-5 text-white">
      <h2 className="text-lg font-bold text-cyan-400 mb-1">AI App Builder Module</h2>
      <p className="text-xs text-slate-400 mb-4">
        Describe any Android application idea in Hindi, Marathi, Bhojpuri or English
      </p>

      <div className="flex space-x-2 mb-4">
        <input
          type="text"
          placeholder='e.g., "Jarvis, mere liye ek offline music player app banao"'
          value={appDescription}
          onChange={(e) => setAppDescription(e.target.value)}
          className="flex-1 bg-slate-800 border border-slate-700 text-xs px-3 py-2 rounded-lg text-white focus:outline-none"
        />
        <button
          onClick={handleBuildPlan}
          disabled={isBuilding || !appDescription}
          className="bg-cyan-600 hover:bg-cyan-500 disabled:opacity-50 text-black font-semibold text-xs px-4 py-2 rounded-lg"
        >
          {isBuilding ? 'Designing...' : 'Generate App'}
        </button>
      </div>

      {generatedCode && (
        <div className="bg-slate-950 border border-slate-800 p-4 rounded-lg">
          <div className="flex justify-between items-center mb-2">
            <span className="text-[10px] font-mono text-cyan-400 uppercase">Architecture Blueprint & Source</span>
            <button
              onClick={() => navigator.clipboard.writeText(generatedCode)}
              className="text-[10px] bg-slate-800 px-2 py-1 rounded text-slate-300 hover:text-white"
            >
              Copy Code
            </button>
          </div>
          <pre className="text-xs text-slate-300 font-mono overflow-x-auto max-h-80 whitespace-pre-wrap">
            {generatedCode}
          </pre>
        </div>
      )}
    </div>
  );
};
import React, { useState } from 'react';
import { AppSettings } from '../types';

export const SocialStudio: React.FC<{ settings: AppSettings; backendUrl: string }> = ({
  settings,
  backendUrl
}) => {
  const [videoTitle, setVideoTitle] = useState('');
  const [description, setDescription] = useState('');
  const [tags, setTags] = useState('JARVIS, AI, Tech, India');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submissionFeedback, setSubmissionFeedback] = useState<string | null>(null);

  const handlePrepareUpload = async () => {
    if (!settings.youtubeUploadEnabled) {
      setSubmissionFeedback('YouTube upload toggle is disabled in Settings. Enable it before publishing.');
      return;
    }

    setIsSubmitting(true);
    setSubmissionFeedback(null);
    try {
      const res = await fetch(`${backendUrl}/api/youtube/prepare-upload`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          title: videoTitle,
          description: description,
          tags: tags.split(',').map((t) => t.trim()),
          privacyStatus: 'unlisted'
        })
      });

      const data = await res.json();
      if (res.ok) {
        setSubmissionFeedback(
          `Payload verified compliant. YouTube Metadata prepared for: "${data.snippet.title}". Requires user OAuth grant on Android device.`
        );
      } else {
        setSubmissionFeedback(`Error: ${data.error}`);
      }
    } catch (err: any) {
      setSubmissionFeedback('Backend unreachable. Make sure Node.js server is online.');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-5 text-white">
      <h2 className="text-lg font-bold text-cyan-400 mb-1">YouTube & Meta Studio Publisher</h2>
      <p className="text-xs text-slate-400 mb-4">Official Google OAuth & Graph API Publishing Assistant</p>

      {/* Platform Publishing Toggles Status Indicator */}
      <div className="grid grid-cols-3 gap-2 text-[11px] font-mono mb-4">
        <div
          className={`p-2 rounded border ${
            settings.youtubeUploadEnabled ? 'border-red-500 bg-red-950/40 text-red-300' : 'border-slate-800 text-slate-500'
          }`}
        >
          YouTube: {settings.youtubeUploadEnabled ? 'AUTHORIZED' : 'OFF'}
        </div>
        <div
          className={`p-2 rounded border ${
            settings.instagramPublishEnabled ? 'border-pink-500 bg-pink-950/40 text-pink-300' : 'border-slate-800 text-slate-500'
          }`}
        >
          Instagram: {settings.instagramPublishEnabled ? 'AUTHORIZED' : 'OFF'}
        </div>
        <div
          className={`p-2 rounded border ${
            settings.facebookPublishEnabled ? 'border-blue-500 bg-blue-950/40 text-blue-300' : 'border-slate-800 text-slate-500'
          }`}
        >
          Facebook: {settings.facebookPublishEnabled ? 'AUTHORIZED' : 'OFF'}
        </div>
      </div>

      <div className="space-y-3">
        <div>
          <label className="text-xs text-slate-300 block mb-1">Video / Post Title</label>
          <input
            type="text"
            placeholder="High-converting SEO Title"
            value={videoTitle}
            onChange={(e) => setVideoTitle(e.target.value)}
            className="w-full bg-slate-800 border border-slate-700 text-xs px-3 py-2 rounded-lg text-white focus:outline-none"
          />
        </div>

        <div>
          <label className="text-xs text-slate-300 block mb-1">Description & Hashtags</label>
          <textarea
            rows={3}
            placeholder="Official description, links, and hashtags..."
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            className="w-full bg-slate-800 border border-slate-700 text-xs px-3 py-2 rounded-lg text-white focus:outline-none"
          />
        </div>

        <div>
          <label className="text-xs text-slate-300 block mb-1">Keywords / SEO Tags (Comma-separated)</label>
          <input
            type="text"
            value={tags}
            onChange={(e) => setTags(e.target.value)}
            className="w-full bg-slate-800 border border-slate-700 text-xs px-3 py-2 rounded-lg text-white focus:outline-none"
          />
        </div>

        <button
          onClick={handlePrepareUpload}
          disabled={isSubmitting || !videoTitle}
          className="w-full bg-red-600 hover:bg-red-500 disabled:opacity-50 text-white font-semibold text-xs py-2.5 rounded-lg transition"
        >
          {isSubmitting ? 'Validating Quotas...' : 'Validate & Prepare YouTube Upload'}
        </button>

        {submissionFeedback && (
          <div className="p-3 bg-slate-950 border border-slate-800 rounded-lg text-xs font-mono text-cyan-300 mt-2">
            {submissionFeedback}
          </div>
        )}
      </div>
    </div>
  );
};
import React, { useState } from 'react';

export const VideoStudio: React.FC<{ backendUrl: string }> = ({ backendUrl }) => {
  const [aspectRatio, setAspectRatio] = useState<'9:16' | '16:9' | '1:1'>('9:16');
  const [scriptPrompt, setScriptPrompt] = useState('');
  const [generatedScript, setGeneratedScript] = useState('');
  const [isGenerating, setIsGenerating] = useState(false);
  const [selectedVideoName, setSelectedVideoName] = useState<string | null>(null);

  const handleGenerateScript = async () => {
    if (!scriptPrompt) return;
    setIsGenerating(true);
    try {
      const res = await fetch(`${backendUrl}/api/chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: `Write a high-converting 30-second video script and caption for YouTube Shorts/Reels about: ${scriptPrompt}. Include scene directions and voiceover text.`
        })
      });
      const data = await res.json();
      setGeneratedScript(data.reply);
    } catch (e) {
      setGeneratedScript('Error generating script. Verify backend connection.');
    } finally {
      setIsGenerating(false);
    }
  };

  const handleVideoSelect = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files && e.target.files[0]) {
      setSelectedVideoName(e.target.files[0].name);
    }
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-5 text-white">
      <h2 className="text-lg font-bold text-cyan-400 mb-1">AI Video Studio & Script Creator</h2>
      <p className="text-xs text-slate-400 mb-4">Export formats for YouTube Shorts, Reels & Horizontal Video</p>

      {/* Aspect Ratio Selector */}
      <div className="mb-4">
        <label className="text-xs font-mono text-slate-300 block mb-1.5">Target Format Ratio</label>
        <div className="grid grid-cols-3 gap-2 text-xs">
          {(['9:16', '16:9', '1:1'] as const).map((ratio) => (
            <button
              key={ratio}
              onClick={() => setAspectRatio(ratio)}
              className={`py-2 rounded-lg font-semibold border ${
                aspectRatio === ratio
                  ? 'border-cyan-400 bg-cyan-950 text-cyan-300'
                  : 'border-slate-800 bg-slate-950 text-slate-400'
              }`}
            >
              {ratio === '9:16' ? '9:16 (Shorts/Reels)' : ratio === '16:9' ? '16:9 (YouTube)' : '1:1 (Feed)'}
            </button>
          ))}
        </div>
      </div>

      {/* Video Import */}
      <div className="mb-4">
        <label className="text-xs font-mono text-slate-300 block mb-1.5">Import Source Video/Image</label>
        <input
          type="file"
          accept="video/*,image/*"
          onChange={handleVideoSelect}
          className="text-xs text-slate-400 file:mr-3 file:py-1.5 file:px-3 file:rounded-md file:border-0 file:text-xs file:font-semibold file:bg-slate-800 file:text-cyan-400 cursor-pointer"
        />
        {selectedVideoName && (
          <p className="text-[11px] text-green-400 mt-1 font-mono">Loaded asset: {selectedVideoName}</p>
        )}
      </div>

      {/* AI Script & Prompt Generation */}
      <div className="mb-4">
        <label className="text-xs font-mono text-slate-300 block mb-1.5">AI Video Script Topic</label>
        <div className="flex space-x-2">
          <input
            type="text"
            placeholder="e.g., Top 5 Smartphone tricks in Hindi"
            value={scriptPrompt}
            onChange={(e) => setScriptPrompt(e.target.value)}
            className="flex-1 bg-slate-800 border border-slate-700 text-xs px-3 py-2 rounded-lg text-white focus:outline-none"
          />
          <button
            onClick={handleGenerateScript}
            disabled={isGenerating || !scriptPrompt}
            className="bg-cyan-600 hover:bg-cyan-500 disabled:opacity-50 text-black font-semibold text-xs px-3 py-2 rounded-lg"
          >
            {isGenerating ? 'Writing...' : 'Generate'}
          </button>
        </div>
      </div>

      {generatedScript && (
        <div className="bg-slate-950 border border-slate-800 p-3 rounded-lg max-h-40 overflow-y-auto text-xs text-slate-300 whitespace-pre-wrap">
          {generatedScript}
        </div>
      )}
    </div>
  );
}
import React, { useState } from 'react';
import { speechEngine } from '../services/speechEngine';
import { SupportedLanguage } from '../types';

interface AlarmProps {
  language: SupportedLanguage;
  assistantName: string;
}

export const AlarmModule: React.FC<AlarmProps> = ({ language, assistantName }) => {
  const [alarmTime, setAlarmTime] = useState('05:00');
  const [isAlarmActive, setIsAlarmActive] = useState(true);
  const [testStatus, setTestStatus] = useState<string | null>(null);

  const getGreeting = () => {
    if (language === 'bho-IN') {
      return `Good morning! Uth jaiye. Subah ho gayil ba! Aaj ke din ke shuruaat kari. Main ${assistantName} aapke sewa mein hazir baani.`;
    }
    if (language === 'mr-IN') {
      return `शुभ सकाळ! उठा, सकाळ झाली आहे. नवीन दिवसाची आनंदाने सुरुवात करा. मी ${assistantName} आपल्या सेवेत हजर आहे.`;
    }
    return `Good morning! Uth jaiye. Subah ho gayi hai! Aaj ke naye din ki shuruaat karein. Main ${assistantName} aapki madad ke liye taiyar hoon.`;
  };

  const handleTestAlarm = () => {
    setTestStatus('Playing 5:00 AM Alarm Audio Sequence...');
    // Synthesize alarm sound tone using Web Audio API (100% Offline)
    const audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.type = 'sawtooth';
    osc.frequency.setValueAtTime(880, audioCtx.currentTime); // A5 note
    gain.gain.setValueAtTime(0.2, audioCtx.currentTime);
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    osc.start();

    setTimeout(() => {
      osc.stop();
      audioCtx.close();
      // Speak the natural regional wake-up greeting
      speechEngine.speak(getGreeting(), language, '', 1.0, 1.0, undefined, () => {
        setTestStatus(null);
      });
    }, 2000);
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-5 text-white">
      <div className="flex items-center justify-between mb-4">
        <div>
          <h2 className="text-lg font-bold text-cyan-400">Daily Alarm & Wakeup Engine</h2>
          <p className="text-xs text-slate-400">Offline-capable AlarmManager with Regional Greeting</p>
        </div>
        <input
          type="checkbox"
          checked={isAlarmActive}
          onChange={(e) => setIsAlarmActive(e.target.checked)}
          className="w-5 h-5 accent-cyan-500 cursor-pointer"
        />
      </div>

      <div className="flex items-center space-x-4 my-4">
        <input
          type="time"
          value={alarmTime}
          onChange={(e) => setAlarmTime(e.target.value)}
          className="bg-slate-800 border border-slate-700 text-2xl font-mono px-3 py-2 rounded-lg text-cyan-300 focus:outline-none"
        />
        <div className="text-xs text-slate-400">
          Default scheduled for <span className="text-white font-bold">5:00 AM</span> every morning.
        </div>
      </div>

      <div className="bg-slate-950 p-3 rounded-lg border border-slate-800 my-3">
        <span className="text-xs text-cyan-500 font-mono">Bhojpuri/Hindi Wakeup Greeting Preview:</span>
        <p className="text-sm italic text-slate-300 mt-1">"{getGreeting()}"</p>
      </div>

      <div className="flex items-center space-x-3 mt-4">
        <button
          onClick={handleTestAlarm}
          className="bg-cyan-600 hover:bg-cyan-500 text-black font-semibold text-xs px-4 py-2.5 rounded-lg transition"
        >
          Test Alarm & Voice
        </button>
        {testStatus && <span className="text-xs text-amber-400 font-mono animate-pulse">{testStatus}</span>}
      </div>
    </div>
  );
};
import React from 'react';

interface OrbProps {
  state: 'idle' | 'listening' | 'thinking' | 'speaking';
  onClick: () => void;
  assistantName: string;
}

export const JarvisOrb: React.FC<OrbProps> = ({ state, onClick, assistantName }) => {
  const getGlowColor = () => {
    switch (state) {
      case 'listening':
        return 'rgba(0, 240, 255, 0.9)'; // Cyan neon
      case 'thinking':
        return 'rgba(255, 170, 0, 0.9)'; // Amber gold
      case 'speaking':
        return 'rgba(0, 255, 128, 0.9)'; // High-energy emerald
      default:
        return 'rgba(0, 150, 255, 0.5)';
    }
  };

  const getStatusLabel = () => {
    switch (state) {
      case 'listening':
        return 'Sun Raha Hoon... (Listening)';
      case 'thinking':
        return 'Soch Raha Hoon... (Thinking)';
      case 'speaking':
        return `${assistantName} Bol Raha Hai...`;
      default:
        return 'Tap To Speak';
    }
  };

  return (
    <div className="flex flex-col items-center justify-center py-6 select-none">
      <div
        onClick={onClick}
        className="relative w-48 h-48 rounded-full flex items-center justify-center cursor-pointer transition-all duration-500 shadow-2xl"
        style={{
          background: 'radial-gradient(circle, #0b1528 20%, #030712 100%)',
          boxShadow: `0 0 45px ${getGlowColor()}, inset 0 0 25px ${getGlowColor()}`
        }}
      >
        {/* Outer Rotating Cyberpunk Ring */}
        <div
          className={`absolute inset-0 rounded-full border-2 border-dashed border-cyan-400 opacity-60 ${
            state === 'thinking' ? 'animate-spin' : 'animate-pulse'
          }`}
          style={{ animationDuration: state === 'thinking' ? '3s' : '8s' }}
        />

        {/* Dynamic Voice Waves */}
        <div className="flex items-center space-x-1.5 z-10">
          {[1, 2, 3, 4, 5, 6].map((i) => (
            <div
              key={i}
              className="w-1.5 bg-cyan-300 rounded-full transition-all duration-150"
              style={{
                height:
                  state === 'speaking'
                    ? `${Math.sin(Date.now() / 100 + i) * 28 + 36}px`
                    : state === 'listening'
                    ? `${(i % 2 === 0 ? 30 : 15)}px`
                    : '6px',
                backgroundColor:
                  state === 'speaking' ? '#00ffaa' : state === 'listening' ? '#00e5ff' : '#0284c7'
              }}
            />
          ))}
        </div>

        {/* Core Arc Reactor Emblem */}
        <div className="absolute w-24 h-24 rounded-full border border-cyan-500 opacity-40 pointer-events-none" />
      </div>

      <span className="mt-4 text-xs font-mono tracking-widest text-cyan-400 uppercase">
        {getStatusLabel()}
      </span>
    </div>
  );
};
{
  "name": "jarvis-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "@google/genai": "^0.1.1",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "googleapis": "^140.0.0",
    "node-fetch": "^3.3.2"
  }
}
jarvis-ai/
├── backend/
│   ├── package.json
│   ├── .env.example
│   └── server.js               # Gemini 2.5 SDK, News Proxy, OAuth proxy
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── types.ts            # Type definitions across all modules
│   │   ├── services/
│   │   │   ├── speechEngine.ts # Multilingual TTS & STT (Hi, En, Mr, Bhoj)
│   │   │   ├── wakeWord.ts     # In-app WebAudio wake word + native bridge
│   │   │   ├── androidBridge.ts# Intent launcher, contacts, flashlight, alarms
│   │   │   └── api.ts          # Backend client for Gemini, News & OAuth
│   │   ├── components/
│   │   │   ├── JarvisOrb.tsx   # Futuristic soundwave arc-reactor indicator
│   │   │   ├── VoiceAssistant.tsx
│   │   │   ├── PhoneControls.tsx
│   │   │   ├── AlarmModule.tsx
│   │   │   ├── IndiaNews.tsx
│   │   │   ├── VideoStudio.tsx
│   │   │   ├── SocialStudio.tsx
│   │   │   ├── AppBuilder.tsx
│   │   │   └── SettingsModal.tsx
│   │   ├── App.tsx             # Central state, audio loop, voice commands
│   │   ├── index.css           # Futuristic dark neon theme
│   │   └── main.tsx
├── android/                    # Android Native Integration Layer
│   ├── AndroidManifest.xml
│   └── java/com/jarvis/ai/
│       ├── MainActivity.kt
│       ├── JarvisNativeBridge.kt
│       └── AlarmReceiver.kt
