# -6-
何を引いたら2面子完成するか？
import React, { useState, useEffect, useRef } from 'react';
import { Trophy, CheckCircle2, XCircle, ChevronRight, Timer, RotateCcw, Play, Zap, ListX, ArrowRight, Coins, Home } from 'lucide-react';

/**
 * 厳選された6枚系多門張データセット（全26問）
 */
const RAW_DATA = `
334556-347
344568-257
223446-35
333445-23456
445678-3469
111233-1234
445567-346
334445-2345
344456-2457
334568-37
134568-27
344567-2458
234446-145
677899-589
356778-469
244566-35
566777-45678
345677-2578
445666-3456
345566-467
455566-4567
456667-3568
134566-26
245679-38
567889-478
133345-236
`;

const parseData = (raw) => {
  return raw
    .trim()
    .split('\n')
    .filter(line => line.includes('-'))
    .map((line, index) => {
      const [q, a] = line.trim().split('-');
      return {
        id: index,
        tiles: q.split('').map(Number),
        answers: a.split('').map(Number)
      };
    });
};

const ALL_PROBLEMS = parseData(RAW_DATA);

const playSound = (type) => {
  try {
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (type === 'correct') {
      const playTone = (freq, startTime, duration) => {
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'sine';
        osc.frequency.setValueAtTime(freq, startTime);
        gain.gain.setValueAtTime(0.1, startTime);
        gain.gain.exponentialRampToValueAtTime(0.01, startTime + duration);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start(startTime);
        osc.stop(startTime + duration);
      };
      const now = audioCtx.currentTime;
      playTone(659.25, now, 0.2); 
      playTone(880.00, now + 0.12, 0.4); 
    } else {
      const osc = audioCtx.createOscillator();
      const gain = audioCtx.createGain();
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(220, audioCtx.currentTime);
      osc.frequency.linearRampToValueAtTime(110, audioCtx.currentTime + 0.3);
      gain.gain.setValueAtTime(0.05, audioCtx.currentTime);
      gain.gain.linearRampToValueAtTime(0, audioCtx.currentTime + 0.3);
      osc.connect(gain);
      gain.connect(audioCtx.destination);
      osc.start();
      osc.stop(audioCtx.currentTime + 0.3);
    }
  } catch (e) { console.error("Audio error", e); }
};

export default function App() {
  const [mode, setMode] = useState(null); 
  const [problemStack, setProblemStack] = useState([]);
  const [currentIdx, setCurrentIdx] = useState(0);
  const [currentQuestion, setCurrentQuestion] = useState(null);
  const [selectedAnswers, setSelectedAnswers] = useState([]);
  const [isSubmitted, setIsSubmitted] = useState(false);
  const [score, setScore] = useState(0);
  const [isFinished, setIsFinished] = useState(false);
  const [lastCorrect, setLastCorrect] = useState(null);
  const [wrongAnswers, setWrongAnswers] = useState([]); 
  const [startTime, setStartTime] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [bestScore, setBestScore] = useState(null);
  const [isNewRecord, setIsNewRecord] = useState(false);

  const timerRef = useRef(null);
  const autoNextRef = useRef(null);

  useEffect(() => {
    if (mode) {
      const saved = localStorage.getItem(`mahjong_best_${mode}`);
      setBestScore(saved ? parseInt(saved) : null);
    }
  }, [mode]);

  const startNewGame = (selectedMode) => {
    const shuffled = [...ALL_PROBLEMS].sort(() => Math.random() - 0.5);
    initSession(shuffled, selectedMode);
  };

  const startRevengeGame = () => {
    const shuffled = [...wrongAnswers].sort(() => Math.random() - 0.5);
    initSession(shuffled, mode);
  };

  const initSession = (stack, selectedMode) => {
    setProblemStack(stack);
    setCurrentIdx(0);
    setScore(0);
    setWrongAnswers([]);
    setIsFinished(false);
    setIsNewRecord(false);
    setElapsedTime(0);
    setStartTime(Date.now());
    setMode(selectedMode);
    setupQuestion(stack[0]);
  };

  const setupQuestion = (q) => {
    if (!q) return;
    setCurrentQuestion({
      ...q,
      displayTiles: [...q.tiles].sort((a, b) => a - b),
      correctSet: [...new Set(q.answers)].sort((a, b) => a - b)
    });
    setSelectedAnswers([]);
    setIsSubmitted(false);
    setLastCorrect(null);
  };

  useEffect(() => {
    if (startTime && !isFinished && mode) {
      timerRef.current = setInterval(() => {
        setElapsedTime(Math.floor((Date.now() - startTime) / 1000));
      }, 1000);
    } else {
      clearInterval(timerRef.current);
    }
    return () => clearInterval(timerRef.current);
  }, [startTime, isFinished, mode]);

  useEffect(() => {
    if (mode === 'timeattack' && isSubmitted) {
      const delay = lastCorrect ? 150 : 1000;
      autoNextRef.current = setTimeout(() => {
        nextQuestion();
      }, delay);
    }
    return () => clearTimeout(autoNextRef.current);
  }, [isSubmitted, mode, lastCorrect]);

  const toggleNum = (num) => {
    if (isSubmitted) return;
    setSelectedAnswers(prev => 
      prev.includes(num) 
        ? prev.filter(n => n !== num).sort((a, b) => a - b) 
        : [...prev, num].sort((a, b) => a - b)
    );
  };

  const handleCheck = () => {
    const isCorrect = JSON.stringify(selectedAnswers) === JSON.stringify(currentQuestion.correctSet);
    setLastCorrect(isCorrect);
    if (isCorrect) {
      setScore(s => s + 1);
      if (mode === 'timeattack') playSound('correct');
    } else {
      setWrongAnswers(prev => [...prev, currentQuestion]);
      if (mode === 'timeattack') playSound('wrong');
    }
    setIsSubmitted(true);
  };

  const nextQuestion = () => {
    const next = currentIdx + 1;
    if (next < problemStack.length) {
      setCurrentIdx(next);
      setupQuestion(problemStack[next]);
    } else {
      finishGame();
    }
  };

  const finishGame = () => {
    setIsFinished(true);
    const pts = calculateTotalPoints().total;
    const savedBest = localStorage.getItem(`mahjong_best_${mode}`);
    if (!savedBest || pts > parseInt(savedBest)) {
      localStorage.setItem(`mahjong_best_${mode}`, pts.toString());
      setIsNewRecord(true);
      setBestScore(pts);
    }
  };

  const formatTime = (seconds) => {
    const m = Math.floor(seconds / 60);
    const s = seconds % 60;
    return `${m}分${s.toString().padStart(2, '0')}秒`;
  };

  const calculateTotalPoints = () => {
    const correctPoints = score * 10;
    const wrongCount = problemStack.length - score;
    const wrongPoints = wrongCount * 10;
    const timePenalty = Math.max(0, elapsedTime - 30);
    const total = correctPoints - wrongPoints - timePenalty;
    return { total, correctPoints, wrongPoints, timePenalty };
  };

  const getRankInfo = (pts) => {
    if (pts >= 250) return { rank: 'SS', comment: 'エクセレント！！！人外のスピードです！', style: 'text-transparent bg-clip-text bg-gradient-to-r from-red-500 via-orange-500 via-yellow-500 via-green-500 via-blue-500 via-purple-500 to-pink-500 animate-pulse' };
    if (pts >= 230) return { rank: 'S', comment: '素晴らしい！6枚系を手足同然に使いこなしています。', style: 'text-yellow-500' };
    if (pts >= 200) return { rank: 'A', comment: 'お見事！更なる速度UPを目指しましょう。', style: 'text-red-600' };
    if (pts >= 150) return { rank: 'B', comment: '安定感がありますね。', style: 'text-green-600' };
    if (pts >= 100) return { rank: 'C', comment: '6枚系ともっと仲良くなりましょう。', style: 'text-blue-600' };
    return { rank: 'D', comment: '６枚系が仲間になりたそうにコチラを見ている。', style: 'text-slate-400' };
  };

  if (!mode) {
    return (
      <div className="min-h-screen bg-slate-100 flex flex-col items-center p-4 sm:p-8 font-sans text-slate-900">
        <div className="max-w-md w-full bg-white rounded-3xl shadow-2xl overflow-hidden border border-slate-200 p-8 text-center space-y-8">
          <div className="space-y-2">
            <h1 className="text-3xl font-black text-indigo-900 flex justify-center items-center gap-2">
              <Trophy className="text-yellow-500" />
              6枚系ドリル
            </h1>
            <p className="text-slate-500 font-bold font-mono tracking-tighter uppercase">Mahjong Training Vol.1</p>
          </div>
          <div className="space-y-4">
            <button onClick={() => startNewGame('practice')} className="w-full group p-6 bg-white border-2 border-indigo-100 rounded-3xl hover:border-indigo-500 hover:bg-indigo-50 transition-all flex items-center gap-4 text-left shadow-sm">
              <div className="bg-indigo-100 p-3 rounded-2xl group-hover:bg-indigo-600 group-hover:text-white transition-colors"><Play size={24} /></div>
              <div>
                <div className="font-black text-xl text-slate-800">練習モード</div>
                <div className="text-sm text-slate-500">回答をじっくり確認して進む</div>
              </div>
            </button>
            <button onClick={() => startNewGame('timeattack')} className="w-full group p-6 bg-white border-2 border-orange-100 rounded-3xl hover:border-orange-500 hover:bg-orange-50 transition-all flex items-center gap-4 text-left shadow-sm">
              <div className="bg-orange-100 p-3 rounded-2xl group-hover:bg-orange-600 group-hover:text-white transition-colors"><Zap size={24} /></div>
              <div>
                <div className="font-black text-xl text-slate-800">タイムアタック</div>
                <div className="text-sm text-slate-500">一瞬で判定して自動で次へ</div>
              </div>
            </button>
          </div>
        </div>
      </div>
    );
  }

  if (isFinished) {
    const accuracy = Math.round((score / problemStack.length) * 100);
    const pts = calculateTotalPoints();
    const rankInfo = getRankInfo(pts.total);

    return (
      <div className="min-h-screen bg-slate-100 flex flex-col items-center p-4 sm:p-8 font-sans text-slate-900 overflow-y-auto">
        <div className="max-w-md w-full bg-white rounded-3xl shadow-2xl overflow-hidden border border-slate-200 p-8 text-center space-y-6">
          <div className="flex justify-center"><div className="bg-indigo-100 p-4 rounded-full"><Trophy className="text-yellow-500 w-12 h-12" /></div></div>
          
          <div className="space-y-1">
            <h2 className="text-3xl font-black text-indigo-900 uppercase tracking-tighter">Result</h2>
            <div className="text-xs font-bold bg-slate-100 py-1 px-4 rounded-full inline-block text-slate-500 tracking-widest uppercase">
              {mode === 'practice' ? '練習モード' : 'タイムアタック'}
            </div>
          </div>

          <div className="py-4 border-y border-slate-100 space-y-3">
            <div className={`text-8xl font-black italic tracking-tighter leading-none ${rankInfo.style}`}>{rankInfo.rank}</div>
            <p className="text-sm font-bold text-slate-800 leading-relaxed px-4 break-keep">{rankInfo.comment}</p>
          </div>

          <div className="bg-slate-900 text-white rounded-3xl p-6 shadow-inner space-y-2 relative overflow-hidden text-left">
             <div className="absolute top-0 right-0 p-4 opacity-10"><Coins size={80}/></div>
             <div className="flex justify-between items-center">
               <p className="text-[10px] font-bold opacity-60 tracking-widest uppercase">Total Score</p>
               {isNewRecord && (
                 <span className="bg-red-600 text-[10px] px-2 py-0.5 rounded font-black animate-bounce">自己ベスト更新！</span>
               )}
             </div>
             <div className="text-6xl font-black tracking-tighter tabular-nums">{pts.total}<span className="text-xl ml-1 font-bold">pt</span></div>
             <div className="flex justify-between items-center pt-3 border-t border-white/10 mt-2">
                <div className="flex gap-4 text-[10px] font-bold opacity-60">
                  <span>正答 +{pts.correctPoints}</span>
                  <span>誤答 -{pts.wrongPoints}</span>
                  <span>時間 -{pts.timePenalty}</span>
                </div>
                <div className="text-[10px] font-bold text-yellow-500">
                  BEST: {bestScore}pt
                </div>
             </div>
          </div>

          <div className="grid grid-cols-2 gap-4 text-left">
            <div className="bg-slate-50 p-4 rounded-2xl border border-slate-200">
              <span className="text-[10px] font-bold text-slate-400 block mb-1 uppercase tracking-tighter">Accuracy</span>
              <div className="text-2xl font-black text-indigo-600">{accuracy}%</div>
              <div className="text-[10px] text-slate-500 font-bold">{score} / {problemStack.length}</div>
            </div>
            <div className="bg-slate-50 p-4 rounded-2xl border border-slate-200">
              <span className="text-[10px] font-bold text-slate-400 block mb-1 uppercase tracking-tighter">Clear Time</span>
              <div className="text-2xl font-black text-indigo-600">{formatTime(elapsedTime)}</div>
            </div>
          </div>

          <div className="flex flex-col gap-3 pt-4">
            {wrongAnswers.length > 0 && (
              <button onClick={startRevengeGame} className="w-full py-4 bg-orange-500 text-white rounded-2xl font-black text-lg hover:bg-orange-600 shadow-lg flex items-center justify-center gap-2 transition-all active:scale-95">
                <RotateCcw size={20} /> ミスした問題のみ解き直し
              </button>
            )}
            <button onClick={() => setMode(null)} className="w-full py-4 bg-indigo-600 text-white rounded-2xl font-black text-lg hover:bg-indigo-700 shadow-xl flex items-center justify-center gap-2 transition-all active:scale-95">
              タイトルへ戻る
            </button>
          </div>
        </div>
      </div>
    );
  }

  if (!currentQuestion) return null;

  return (
    <div className="min-h-screen bg-slate-50 flex flex-col items-center p-4 sm:p-8 font-sans text-slate-900">
      <div className="max-w-md w-full bg-white rounded-3xl shadow-2xl overflow-hidden border border-slate-200">
        <div className={`${mode === 'timeattack' ? 'bg-orange-600' : 'bg-indigo-900'} p-6 text-white transition-colors duration-500`}>
          <div className="flex justify-between items-start mb-1">
            <div className="flex flex-col">
              <h1 className="text-xl font-black tracking-wider flex items-center gap-2">
                {mode === 'timeattack' ? <Zap className="text-yellow-300 w-5 h-5" /> : <Trophy className="text-yellow-400 w-5 h-5" />}
                {mode === 'timeattack' ? 'タイムアタック' : '練習モード'}
              </h1>
              <p className="text-[10px] font-bold text-white/70 mt-0.5 tracking-tight">何を引いたら2面子完成？</p>
            </div>
            <div className="flex items-center gap-2">
              <button 
                onClick={() => setMode(null)}
                className="p-2 bg-white/10 hover:bg-white/20 rounded-lg transition-colors flex items-center gap-1 group"
              >
                <Home size={18} />
                <span className="text-[10px] font-bold hidden group-hover:inline">終了</span>
              </button>
              <div className="bg-black/20 px-3 py-1 rounded-full text-xs font-bold font-mono">
                {formatTime(elapsedTime)}
              </div>
              <div className="bg-white/10 px-3 py-1 rounded-full text-xs font-bold">
                {currentIdx + 1} / {problemStack.length}
              </div>
            </div>
          </div>
        </div>

        <div className={`py-12 flex justify-center items-center gap-0 border-b-4 border-slate-200 ${mode === 'timeattack' ? 'bg-orange-50' : 'bg-indigo-50'}`}>
          {currentQuestion.displayTiles.map((num, idx) => (
            <div key={idx} className="w-12 h-16 bg-white border-2 border-slate-300 rounded-lg flex flex-col items-center justify-center shadow-md relative text-slate-800" style={{ borderBottomWidth: '6px' }}>
              <span className="text-3xl font-black">{num}</span>
            </div>
          ))}
        </div>

        <div className="p-6 space-y-6">
          <div className="grid grid-cols-3 gap-3">
            {[1, 2, 3, 4, 5, 6, 7, 8, 9].map(num => (
              <button
                key={num}
                onClick={() => toggleNum(num)}
                disabled={isSubmitted}
                className={`py-4 rounded-2xl font-black text-2xl transition-all active:scale-95 ${selectedAnswers.includes(num) ? (mode === 'timeattack' ? 'bg-orange-600 text-white shadow-inner translate-y-1' : 'bg-indigo-600 text-white shadow-inner translate-y-1') : 'bg-white border-2 border-slate-200 text-slate-700 hover:border-slate-400 shadow-sm'} ${isSubmitted && currentQuestion.correctSet.includes(num) ? 'ring-4 ring-yellow-400 z-10' : ''}`}
              >
                {num}
              </button>
            ))}
          </div>

          {!isSubmitted ? (
            <button 
              onClick={handleCheck} 
              disabled={selectedAnswers.length === 0} 
              className={`w-full py-5 text-white rounded-2xl font-black text-lg disabled:opacity-30 shadow-lg transition-all active:scale-[0.98] ${mode === 'timeattack' ? 'bg-orange-700 hover:bg-orange-800' : 'bg-slate-900 hover:bg-slate-800'}`}
            >
              回答を確定する
            </button>
          ) : (
            <div className="space-y-4">
              <div className={`p-4 rounded-2xl flex items-center justify-between border-2 ${lastCorrect ? 'bg-green-50 text-green-800 border-green-200' : 'bg-red-50 text-red-800 border-red-200'}`}>
                <div className="flex items-center gap-3">
                  {lastCorrect ? <CheckCircle2 className="w-8 h-8 text-green-600" /> : <XCircle className="w-8 h-8 text-red-600" />}
                  <div>
                    <p className="font-black text-lg leading-tight uppercase tracking-tight">{lastCorrect ? '正解！' : '不正解...'}</p>
                    <p className="text-3xl font-black tracking-[0.2em] leading-none font-mono">{currentQuestion.correctSet.join('')}</p>
                  </div>
                </div>
              </div>
              {mode === 'practice' && (
                <button 
                  onClick={nextQuestion} 
                  className="w-full py-5 bg-indigo-600 text-white rounded-2xl font-black text-lg hover:bg-indigo-700 shadow-xl flex items-center justify-center gap-2 transition-transform active:scale-95"
                >
                  {currentIdx + 1 < problemStack.length ? '次の問題へ' : '結果を見る'} <ChevronRight />
                </button>
              )}
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
