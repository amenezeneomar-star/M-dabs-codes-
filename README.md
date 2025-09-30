# M-dabs-codes-
Ai small biussness 
import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, query, onSnapshot, orderBy, limit } from 'firebase/firestore';
import { setLogLevel } from 'firebase/firestore';

// Set Firebase log level for debugging
setLogLevel('debug');

// --- Global Configuration ---
// These global variables are provided by the execution environment.
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// !!! IMPORTANT: UPDATE THIS ENDPOINT AFTER DEPLOYING YOUR SERVERLESS FUNCTION (File #2)
const BACKEND_ENDPOINT = 'YOUR_SECURE_SERVERLESS_API_ENDPOINT_HERE';

/**
 * Simulates the secure backend call. 
 * In a live environment, this should be a real fetch call to the BACKEND_ENDPOINT.
 */
const callSecureBackend = async (contractText) => {
    
    const systemPrompt = "You are a specialized Legal Analysis AI assistant for small businesses. Your purpose is to review contracts and quickly extract the essential information and flag potential risks and liabilities for the client (the small business owner). Your tone must be formal, objective, and easy for a non-lawyer to understand. You must respond ONLY with the analysis, structured exactly as requested, and ensure all three sections are present. Use **bold text** to highlight key terms or critical points within the sections.";

    const userQuery = `Analyze the following contract text. Provide your analysis structured into three distinct sections using Markdown headings (##) exactly as requested: 1. ## Concise Summary, 2. ## Red Flags and Potential Risks, 3. ## Basic Suggestions and Negotiation Points. CONTRACT TEXT: ${contractText}`;

    // --- Mock Response for Sandbox Testing ---
    const mockResponse = {
        text: `## Concise Summary
This **Service Agreement** is between **Client Co.** and **Vendor Corp.** for cloud hosting services at a monthly fee of **$299** for 99.9% uptime. The initial term is **12 months** and it renews automatically with a **90-day written notice** requirement.

## Red Flags and Potential Risks
- **Indemnification Clause:** The client agrees to indemnify Vendor Corp. against **all damages and legal fees**, which is overly broad and represents a significant liability risk.
- **Service Credits Limit:** Uptime compensation is capped at **10% of the monthly fee**, severely limiting recourse if prolonged downtime occurs.
- **Automatic Renewal:** The 90-day notice is long; failure to provide it locks the client into another **12-month term**.
- **Choice of Law:** The contract stipulates that **Vendor Corp.'s home state law** governs the agreement, forcing the client to potentially litigate in a distant jurisdiction.

## Basic Suggestions and Negotiation Points
- **Limit Indemnification:** Negotiate the indemnification clause to be **mutual** and carve out liability for Vendor Corp.'s **negligence or willful misconduct**.
- **Increase Service Credits:** Request service credits be increased to **25-50%** of the monthly fee for extended outages.
- **Amend Renewal Terms:** Propose changing the automatic renewal to a **30-day notice period** or converting the renewal to **month-to-month** after the initial term.`,
        sources: []
    };

    // Replace this simulation block with the actual secure fetch call in production:
    /*
    const payload = { userQuery, systemPrompt, contractText };
    const response = await fetch(BACKEND_ENDPOINT, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
    });
    if (!response.ok) throw new Error("Backend API call failed.");
    const data = await response.json();
    return data; 
    */
    
    await new Promise(resolve => setTimeout(resolve, 2500)); 
    return mockResponse;
};

// --- Custom Markdown Renderer Component ---
const MarkdownRenderer = ({ markdownText }) => {
    const renderHtml = useMemo(() => {
        if (!markdownText) return '';
        let html = markdownText;

        html = html.replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>');
        html = html.replace(/^## (.*)$/gm, '<h2 class="analysis-heading">$1</h2>');
        html = html.replace(/^(\s*[*+-]\s)(.*)/gm, '<li class="analysis-list-item">$2</li>');
        html = html.replace(/(<li class="analysis-list-item">.*<\/li>)/gms, '<ul class="analysis-list">$1</ul>');
        html = html.replace(/<\/ul>\s*<ul class="analysis-list">/g, ''); 

        html = html.split('\n').map(line => {
            const trimmedLine = line.trim();
            if (trimmedLine && !trimmedLine.match(/^(<h2|<ul|<li)/)) {
                return `<p class="mb-4">${trimmedLine}</p>`;
            }
            return line;
        }).join('\n');
        html = html.replace(/<p class="mb-4"><\/p>/g, '');
        return html;
    }, [markdownText]);

    return <div dangerouslySetInnerHTML={{ __html: renderHtml }} />;
};

// --- Analysis Item Structure ---
const AnalysisItem = ({ data }) => {
    const formattedDate = new Date(data.timestamp).toLocaleDateString('en-US', {
        year: 'numeric', month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit'
    });
    const [isExpanded, setIsExpanded] = useState(false);

    return (
        <div className="bg-white p-4 rounded-xl shadow-md border border-gray-100 hover:shadow-lg transition duration-300 mb-4">
            <div className="flex justify-between items-center cursor-pointer" onClick={() => setIsExpanded(!isExpanded)}>
                <h3 className="text-lg font-bold text-brand-secondary">
                    Analysis from {formattedDate}
                </h3>
                <span className="text-xl text-brand-primary font-extrabold">
                    {isExpanded ? '▲' : '▼'}
                </span>
            </div>
            {isExpanded && (
                <div className="mt-4 pt-4 border-t border-gray-200">
                    <p className="text-gray-600 italic mb-4">Original Text Snippet: "{data.originalText.substring(0, 150)}..."</p>
                    <div className="text-sm text-brand-text">
                        <MarkdownRenderer markdownText={data.analysisResult} />
                    </div>
                </div>
            )}
        </div>
    );
};


// --- Main App Component ---
export default function App() {
    const [contractText, setContractText] = useState('');
    const [analysisResult, setAnalysisResult] = useState(null);
    const [isLoading, setIsLoading] = useState(false);
    const [message, setMessage] = useState(null);
    const [userId, setUserId] = useState(null);
    const [pastAnalyses, setPastAnalyses] = useState([]);

    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    // 1. Firebase Initialization and Authentication
    useEffect(() => {
        if (!firebaseConfig) return;
        
        try {
            const app = initializeApp(firebaseConfig);
            const authInstance = getAuth(app);
            const dbInstance = getFirestore(app);
            setDb(dbInstance);
            setAuth(authInstance);

            const unsubscribe = onAuthStateChanged(authInstance, async (user) => {
                if (user) {
                    setUserId(user.uid);
                } else {
                    if (!initialAuthToken) {
                         await signInAnonymously(authInstance);
                    }
                }
                setIsAuthReady(true);
            });

            if (initialAuthToken) {
                signInWithCustomToken(authInstance, initialAuthToken).catch(e => {
                    console.error("Custom token sign-in failed:", e);
                    signInAnonymously(authInstance);
                });
            } else {
                 signInAnonymously(authInstance);
            }

            return () => unsubscribe();
        } catch (e) {
            console.error("Firebase initialization failed:", e);
            setMessage({ type: 'error', text: "Database connection failed. Check console for details." });
        }
    }, []);

    // 2. Data Persistence (Fetching Past Analyses)
    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        const analysesRef = collection(db, 'artifacts', appId, 'users', userId, 'analyses');
        const q = query(analysesRef, orderBy('timestamp', 'desc'), limit(10));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const analyses = [];
            snapshot.forEach((doc) => {
                analyses.push({ id: doc.id, ...doc.data() });
            });
            setPastAnalyses(analyses);
        }, (error) => {
            console.error("Firestore snapshot error:", error);
            setMessage({ type: 'error', text: "Could not load past analyses." });
        });

        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    // 3. Main Analysis Logic
    const handleAnalyze = async () => {
        const text = contractText.trim();
        setMessage(null);
        setAnalysisResult(null);

        if (!isAuthReady || !db || !userId) {
            setMessage({ type: 'error', text: 'Authentication not ready. Please wait a moment.' });
            return;
        }

        if (text.length < 500) {
            setMessage({ type: 'warning', text: "Contract text must be at least 500 characters for a deep analysis." });
            return;
        }
        
        setIsLoading(true);

        try {
            // NOTE: In production, this will hit the live BACKEND_ENDPOINT
            const data = await callSecureBackend(text); 
            
            const analysesRef = collection(db, 'artifacts', appId, 'users', userId, 'analyses');
            await addDoc(analysesRef, {
                originalText: text,
                analysisResult: data.text,
                timestamp: Date.now(),
            });
            
            setAnalysisResult(data.text);
            setMessage({ type: 'success', text: 'Analysis complete and successfully saved to your history!' });

        } catch (error) {
            console.error("Analysis failed:", error);
            setMessage({ type: 'error', text: `Analysis failed: ${error.message}. Please try again.` });
        } finally {
            setIsLoading(false);
        }
    };

    const displayMessage = message ? (
        <div className={`p-4 text-center rounded-xl mb-6 font-semibold shadow-md ${
            message.type === 'error' ? 'bg-brand-danger/10 text-brand-danger border border-brand-danger' :
            message.type === 'success' ? 'bg-brand-primary/10 text-brand-primary border border-brand-primary' :
            'bg-brand-secondary/10 text-brand-secondary border border-brand-secondary'
        }`}>
            {message.text}
        </div>
    ) : null;

    return (
        <div className="min-h-screen bg-brand-background p-4 sm:p-8 font-sans">
            <div className="container-content mx-auto bg-white shadow-2xl rounded-2xl p-6 md:p-12">
                <header className="text-center mb-8">
                    <h1 className="text-4xl sm:text-5xl font-extrabold text-brand-secondary tracking-tight">Contract Scout <span className="text-brand-primary">Pro</span></h1>
                    <p className="text-gray-600 mt-2 max-w-2xl mx-auto">AI-Powered Risk Analysis for Small Business Contracts. User ID: <span className="font-mono text-xs p-1 bg-gray-100 rounded">{userId || 'Loading...'}</span></p>
                </header>

                {displayMessage}

                {/* Input Section */}
                <div className="mb-6 p-6 bg-gray-50 rounded-xl border border-gray-200 shadow-inner">
                    <label htmlFor="contractText" className="block text-xl font-bold text-brand-text mb-3">
                        Paste Contract Text Below
                    </label>
                    <textarea 
                        id="contractText" 
                        rows="12" 
                        value={contractText}
                        onChange={(e) => setContractText(e.target.value)}
                        className="w-full p-4 border-2 border-brand-secondary/30 rounded-lg focus:ring-brand-primary focus:border-brand-primary transition duration-300 shadow-md text-base" 
                        placeholder="Paste the full text of your legal document here (e.g., NDA, MSA, Lease)."
                        disabled={isLoading}
                    />
                </div>

                {/* Action Button */}
                <div className="flex justify-center mb-10">
                    <button 
                        onClick={handleAnalyze}
                        disabled={isLoading || contractText.length < 500}
                        className="bg-brand-primary hover:bg-green-600 text-white font-extrabold py-3 px-8 rounded-full transition duration-300 ease-in-out shadow-lg shadow-brand-primary/40 transform hover:scale-105 active:scale-95 flex items-center justify-center text-lg disabled:opacity-50 disabled:shadow-none disabled:transform-none">
                        {isLoading ? (
                            <>
                                <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                                Analyzing Contract...
                            </>
                        ) : 'Analyze Contract & Save Report'}
                    </button>
                </div>
                
                {/* Current Analysis Output */}
                {analysisResult && (
                    <div className="mt-10">
                        <div className="bg-white p-6 sm:p-8 rounded-xl border-4 border-brand-secondary/50 shadow-2xl shadow-brand-secondary/10">
                            <div className="text-center mb-6">
                                <span className="inline-block text-3xl font-bold text-brand-secondary border-b-4 border-brand-primary pb-1">Current Analysis Report</span>
                            </div>
                            <div id="analysisOutput" className="text-brand-text leading-relaxed text-lg">
                                <MarkdownRenderer markdownText={analysisResult} />
                            </div>

                            <div className="mt-8 pt-4 border-t border-gray-200 text-center">
                                <p className="text-sm font-semibold text-brand-danger">⚠️ IMPORTANT LEGAL DISCLAIMER ⚠️</p>
                                <p className="text-sm text-gray-500 mt-1">This AI analysis is NOT a substitute for professional legal counsel. Consult a qualified lawyer before signing any document.</p>
                            </div>
                        </div>
                    </div>
                )}

                {/* Past Analyses History */}
                <div className="mt-12 pt-8 border-t border-gray-200">
                    <h2 className="text-3xl font-bold text-brand-secondary mb-6 text-center">Past Analyses History</h2>
                    {isAuthReady && !userId && <p className="text-center text-gray-500">History loading...</p>}
                    {pastAnalyses.length === 0 && isAuthReady && userId && <p className="text-center text-gray-500 italic">No saved analysis reports found yet.</p>}
                    
                    <div className="grid grid-cols-1 gap-4">
                        {pastAnalyses.map(analysis => (
                            <AnalysisItem key={analysis.id} data={analysis} />
                        ))}
                    </div>
                </div>

            </div>
            {/* Custom CSS for Markdown Rendering in React */}
            <style jsx="true">{`
                .analysis-heading {
                    font-size: 1.625rem;
                    font-weight: 700;
                    margin-top: 2rem;
                    margin-bottom: 0.75rem;
                    padding-bottom: 0.5rem;
                    border-bottom: 3px solid #4f46e5; /* brand-secondary */
                    color: #4f46e5;
                    line-height: 1.2;
                }
                .analysis-list {
                    list-style-type: none;
                    margin-left: 0;
                    padding-left: 0;
                }
                .analysis-list-item {
                    position: relative;
                    margin-bottom: 1rem;
                    padding-left: 1.75rem;
                    line-height: 1.6;
                }
                /* Red Flags (2nd Heading) - Warning sign */
                .analysis-heading:nth-of-type(2) + .analysis-list .analysis-list-item::before {
                    content: '🚨';
                    position: absolute;
                    left: 0;
                    top: 0;
                }
                /* Suggestions (3rd Heading) - Checkmark */
                .analysis-heading:nth-of-type(3) + .analysis-list .analysis-list-item::before {
                    content: '✅';
                    position: absolute;
                    left: 0;
                    top: 0;
                }
                strong {
                    font-weight: 800;
                    color: #1f2937; /* brand-text */
                }
            `}</style>
        </div>
    );
}
/**
 * Serverless function logic for Contract Analysis.
 * * This file is critical for security and must be deployed to a private server 
 * to shield the GEMINI_API_KEY from the client (browser).
 * * Dependencies (Node.js/Express environment assumed):
 * - node-fetch (if running in a non-standard Node environment)
 * - Express or equivalent serverless function wrapper
 */

// NOTE: The key should be retrieved from a secure environment variable.
const GEMINI_API_KEY = process.env.GEMINI_API_KEY || "YOUR_SECRET_KEY_GOES_HERE"; 
const API_URL = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent";

/**
 * Handles the incoming request from the front-end client and executes the AI analysis.
 * In a real environment, this function would be wrapped by an Express route or Lambda handler.
 * @param {object} req - The request object (expected to have body.userQuery and body.systemPrompt).
 * @param {object} res - The response object to send back to the front-end.
 */
async function handleAnalysisRequest(req, res) {
    // Basic validation of input from the client
    const { userQuery, systemPrompt } = req.body;

    if (!userQuery || !systemPrompt) {
        return res.status(400).json({ error: "Missing required query parameters for analysis." });
    }

    // Define the payload for the Gemini API call
    const payload = {
        contents: [{ parts: [{ text: userQuery }] }],
        systemInstruction: { parts: [{ text: systemPrompt }] },
    };

    const MAX_RETRIES = 3;

    for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
        try {
            // Securely call the Gemini API using the protected key
            const response = await fetch(`${API_URL}?key=${GEMINI_API_KEY}`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            if (!response.ok) {
                // Implement Exponential Backoff for 429 (rate limiting) errors
                if (response.status === 429 && attempt < MAX_RETRIES) {
                    const delay = Math.pow(2, attempt) * 1000;
                    await new Promise(resolve => setTimeout(resolve, delay));
                    continue; // Retry
                }
                throw new Error(`Gemini API call failed with status: ${response.status}`);
            }

            const apiResult = await response.json();
            const generatedText = apiResult.candidates?.[0]?.content?.parts?.[0]?.text;

            if (!generatedText) {
                throw new Error("Gemini API response was empty or malformed.");
            }
            
            // Return the clean, generated text to the front-end
            const result = {
                text: generatedText,
                sources: apiResult.candidates[0].groundingMetadata?.groundingAttributions || [],
            };
            return res.status(200).json(result);

        } catch (error) {
            console.error(`Analysis attempt ${attempt} failed:`, error);
            if (attempt === MAX_RETRIES) {
                // If max retries reached, send final 500 error to client
                return res.status(500).json({ error: "Internal server error: AI analysis failed." });
            }
        }
    }
}

// Example export for a typical serverless function environment (e.g., Netlify/Vercel)
// module.exports = async (req, res) => {
//     if (req.method === 'POST') {
//         return handleAnalysisRequest(req, res);
//     }
//     res.status(405).send('Method Not Allowed');
// };


