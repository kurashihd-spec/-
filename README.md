import { useState, useEffect, useCallback, useRef } from "react";

const COLS = 10;
const ROWS = 20;

const TETROMINOES = {
  I: { shape: [[1,1,1,1]], color: "#00f5ff" },
  O: { shape: [[1,1],[1,1]], color: "#ffe600" },
  T: { shape: [[0,1,0],[1,1,1]], color: "#c800ff" },
  S: { shape: [[0,1,1],[1,1,0]], color: "#00ff41" },
  Z: { shape: [[1,1,0],[0,1,1]], color: "#ff2a2a" },
  J: { shape: [[1,0,0],[1,1,1]], color: "#0047ff" },
  L: { shape: [[0,0,1],[1,1,1]], color: "#ff8800" },
};

const PIECES = Object.values(TETROMINOES);
const SCORE_TABLE = [0, 100, 300, 500, 800];

function randomPiece() { return PIECES[Math.floor(Math.random() * PIECES.length)]; }
function rotate(shape) { return shape[0].map((_, i) => shape.map(row => row[i]).reverse()); }
function createBoard() { return Array.from({ length: ROWS }, () => Array(COLS).fill(null)); }

function isValid(board, shape, pos) {
  for (let r = 0; r < shape.length; r++)
    for (let c = 0; c < shape[r].length; c++) {
      if (!shape[r][c]) continue;
      const nr = pos.r + r, nc = pos.c + c;
      if (nr < 0 || nr >= ROWS || nc < 0 || nc >= COLS) return false;
      if (board[nr][nc]) return false;
    }
  return true;
}

function place(board, shape, pos, color) {
  const nb = board.map(r => [...r]);
  for (let r = 0; r < shape.length; r++)
    for (let c = 0; c < shape[r].length; c++)
      if (shape[r][c]) nb[pos.r + r][pos.c + c] = color;
  return nb;
}

function clearLines(board) {
  const kept = board.filter(row => row.some(c => !c));
  const cleared = ROWS - kept.length;
  const empty = Array.from({ length: cleared }, () => Array(COLS).fill(null));
  return { board: [...empty, ...kept], cleared };
}

// Touch button component
function TouchBtn({ onPress, children, style = {}, color = "#00f5ff" }) {
  const [active, setActive] = useState(false);
  const intervalRef = useRef(null);

  const start = useCallback(() => {
    setActive(true);
    onPress();
    intervalRef.current = setInterval(onPress, 120);
  }, [onPress]);

  const end = useCallback(() => {
    setActive(false);
    clearInterval(intervalRef.current);
  }, []);

  return (
    <button
      onTouchStart={e => { e.preventDefault(); start(); }}
      onTouchEnd={e => { e.preventDefault(); end(); }}
      onMouseDown={start}
      onMouseUp={end}
      onMouseLeave={end}
      style={{
        background: active ? `rgba(${hexToRgb(color)},0.3)` : `rgba(${hexToRgb(color)},0.08)`,
        border: `2px solid ${color}`,
        color,
        borderRadius: 12,
        fontSize: 22,
        display: "flex", alignItems: "center", justifyContent: "center",
        cursor: "pointer",
        boxShadow: active ? `0 0 20px ${color}` : `0 0 8px rgba(${hexToRgb(color)},0.3)`,
        transition: "all 0.08s",
        WebkitTapHighlightColor: "transparent",
        userSelect: "none",
        ...style,
      }}
    >
      {children}
    </button>
  );
}

function hexToRgb(hex) {
  const r = parseInt(hex.slice(1,3),16);
  const g = parseInt(hex.slice(3,5),16);
  const b = parseInt(hex.slice(5,7),16);
  return `${r},${g},${b}`;
}

export default function Tetris() {
  const [board, setBoard] = useState(createBoard());
  const [current, setCurrent] = useState(null);
  const [pos, setPos] = useState({ r: 0, c: 3 });
  const [next, setNext] = useState(randomPiece());
  const [score, setScore] = useState(0);
  const [lines, setLines] = useState(0);
  const [level, setLevel] = useState(1);
  const [gameOver, setGameOver] = useState(false);
  const [started, setStarted] = useState(false);
  const [paused, setPaused] = useState(false);
  const [flash, setFlash] = useState(false);
  const [blockSize, setBlockSize] = useState(28);

  const boardRef = useRef(board);
  const currentRef = useRef(current);
  const posRef = useRef(pos);
  const nextRef = useRef(next);
  const levelRef = useRef(level);
  boardRef.current = board;
  currentRef.current = current;
  posRef.current = pos;
  nextRef.current = next;
  levelRef.current = level;

  // Responsive block size
  useEffect(() => {
    const update = () => {
      const maxH = window.innerHeight * 0.55;
      const maxW = window.innerWidth * 0.56;
      const bs = Math.floor(Math.min(maxH / ROWS, maxW / COLS));
      setBlockSize(Math.max(16, Math.min(32, bs)));
    };
    update();
    window.addEventListener("resize", update);
    return () => window.removeEventListener("resize", update);
  }, []);

  const spawnPiece = useCallback((nb, nxt) => {
    const piece = nxt || nextRef.current;
    const newNext = randomPiece();
    const startPos = { r: 0, c: Math.floor((COLS - piece.shape[0].length) / 2) };
    if (!isValid(nb, piece.shape, startPos)) { setGameOver(true); return; }
    setCurrent(piece);
    setPos(startPos);
    setNext(newNext);
  }, []);

  const lockPiece = useCallback(() => {
    const b = boardRef.current, cur = currentRef.current, p = posRef.current;
    if (!cur) return;
    const nb = place(b, cur.shape, p, cur.color);
    const { board: cb, cleared } = clearLines(nb);
    if (cleared > 0) { setFlash(true); setTimeout(() => setFlash(false), 200); }
    setScore(s => s + SCORE_TABLE[cleared] * levelRef.current);
    setLines(l => { const nl = l + cleared; setLevel(Math.floor(nl / 10) + 1); return nl; });
    setBoard(cb);
    setCurrent(null);
    setTimeout(() => spawnPiece(cb, null), 0);
  }, [spawnPiece]);

  const moveDown = useCallback(() => {
    const cur = currentRef.current, p = posRef.current, b = boardRef.current;
    if (!cur) return;
    const np = { ...p, r: p.r + 1 };
    if (isValid(b, cur.shape, np)) setPos(np); else lockPiece();
  }, [lockPiece]);

  const moveLeft = useCallback(() => {
    const cur = currentRef.current, p = posRef.current, b = boardRef.current;
    if (!cur) return;
    const np = { ...p, c: p.c - 1 };
    if (isValid(b, cur.shape, np)) setPos(np);
  }, []);

  const moveRight = useCallback(() => {
    const cur = currentRef.current, p = posRef.current, b = boardRef.current;
    if (!cur) return;
    const np = { ...p, c: p.c + 1 };
    if (isValid(b, cur.shape, np)) setPos(np);
  }, []);

  const doRotate = useCallback(() => {
    const cur = currentRef.current, p = posRef.current, b = boardRef.current;
    if (!cur) return;
    const rotated = rotate(cur.shape);
    if (isValid(b, rotated, p)) setCurrent({ ...cur, shape: rotated });
  }, []);

  const hardDrop = useCallback(() => {
    const cur = currentRef.current, p = posRef.current, b = boardRef.current;
    if (!cur) return;
    let np = { ...p };
    while (isValid(b, cur.shape, { ...np, r: np.r + 1 })) np.r++;
    setPos(np);
    setTimeout(lockPiece, 0);
  }, [lockPiece]);

  useEffect(() => {
    if (!started || gameOver || paused || !current) return;
    const speed = Math.max(100, 600 - (level - 1) * 50);
    const id = setInterval(moveDown, speed);
    return () => clearInterval(id);
  }, [started, gameOver, paused, current, level, moveDown]);

  // Keyboard support
  const handleKey = useCallback((e) => {
    if (!started || gameOver || paused) return;
    if (e.key === "ArrowLeft") moveLeft();
    else if (e.key === "ArrowRight") moveRight();
    else if (e.key === "ArrowDown") moveDown();
    else if (e.key === "ArrowUp" || e.key === "x") doRotate();
    else if (e.key === " ") hardDrop();
    else if (e.key === "p" || e.key === "Escape") setPaused(v => !v);
  }, [started, gameOver, paused, moveLeft, moveRight, moveDown, doRotate, hardDrop]);

  useEffect(() => {
    window.addEventListener("keydown", handleKey);
    return () => window.removeEventListener("keydown", handleKey);
  }, [handleKey]);

  // Swipe detection
  const touchStart = useRef(null);
  const handleTouchStart = useCallback((e) => {
    touchStart.current = { x: e.touches[0].clientX, y: e.touches[0].clientY };
  }, []);
  const handleTouchEnd = useCallback((e) => {
    if (!touchStart.current || !started || gameOver || paused) return;
    const dx = e.changedTouches[0].clientX - touchStart.current.x;
    const dy = e.changedTouches[0].clientY - touchStart.current.y;
    if (Math.abs(dx) < 10 && Math.abs(dy) < 10) { doRotate(); return; }
    if (Math.abs(dy) > Math.abs(dx) && dy > 40) { hardDrop(); return; }
  }, [started, gameOver, paused, doRotate, hardDrop]);

  const startGame = () => {
    const nb = createBoard();
    setBoard(nb); setScore(0); setLines(0); setLevel(1);
    setGameOver(false); setPaused(false); setStarted(true);
    const first = randomPiece(), sNext = randomPiece();
    setNext(sNext);
    const startPos = { r: 0, c: Math.floor((COLS - first.shape[0].length) / 2) };
    setCurrent(first); setPos(startPos);
  };

  // Ghost piece
  let ghostPos = pos;
  if (current && started) {
    let gp = { ...pos };
    while (isValid(board, current.shape, { ...gp, r: gp.r + 1 })) gp.r++;
    ghostPos = gp;
  }

  const displayBoard = board.map(r => [...r]);
  if (current && started && !gameOver) {
    for (let r = 0; r < current.shape.length; r++)
      for (let c = 0; c < current.shape[r].length; c++) {
        if (!current.shape[r][c]) continue;
        const gr = ghostPos.r + r, gc = ghostPos.c + c;
        if (gr >= 0 && gr < ROWS && gc >= 0 && gc < COLS && !displayBoard[gr][gc])
          displayBoard[gr][gc] = "ghost";
        const pr = pos.r + r, pc = pos.c + c;
        if (pr >= 0 && pr < ROWS && pc >= 0 && pc < COLS)
          displayBoard[pr][pc] = current.color;
      }
  }

  const renderMini = (piece) => {
    if (!piece) return null;
    return piece.shape.map((row, r) => (
      <div key={r} style={{ display: "flex" }}>
        {row.map((c, ci) => (
          <div key={ci} style={{
            width: 12, height: 12, margin: 1,
            background: c ? piece.color : "transparent",
            borderRadius: 2,
            boxShadow: c ? `0 0 4px ${piece.color}` : "none",
          }} />
        ))}
      </div>
    ));
  };

  const BK = blockSize;

  return (
    <div style={{
      height: "100dvh",
      background: "#0a0a0f",
      display: "flex",
      flexDirection: "column",
      alignItems: "center",
      justifyContent: "space-between",
      fontFamily: "'Courier New', monospace",
      backgroundImage: "radial-gradient(ellipse at 50% 0%, #1a0030 0%, #0a0a0f 70%)",
      overflow: "hidden",
      paddingBottom: "env(safe-area-inset-bottom)",
    }}
      onTouchStart={handleTouchStart}
      onTouchEnd={handleTouchEnd}
    >
      {/* Scanlines */}
      <div style={{
        position: "fixed", inset: 0, pointerEvents: "none", zIndex: 10,
        backgroundImage: "repeating-linear-gradient(0deg, transparent, transparent 2px, rgba(0,0,0,0.12) 2px, rgba(0,0,0,0.12) 4px)",
      }} />

      {/* Top bar */}
      <div style={{
        width: "100%", display: "flex", justifyContent: "space-between",
        alignItems: "center", padding: "8px 16px", boxSizing: "border-box",
        borderBottom: "1px solid rgba(255,255,255,0.06)",
      }}>
        <div style={{ display: "flex", gap: 16 }}>
          <Stat label="SCORE" value={score.toString().padStart(7,"0")} color="#ffe600" />
          <Stat label="LV" value={level} color="#00f5ff" />
          <Stat label="LINES" value={lines} color="#00ff41" />
        </div>
        <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
          <div style={{ fontSize: 9, color: "#555", letterSpacing: 2 }}>NEXT</div>
          <div style={{ display: "flex", flexDirection: "column" }}>{renderMini(next)}</div>
        </div>
        <button
          onClick={() => started && !gameOver && setPaused(v => !v)}
          style={{
            background: "transparent", border: "1px solid #555",
            color: "#888", fontSize: 11, letterSpacing: 2,
            padding: "4px 10px", borderRadius: 4, cursor: "pointer",
          }}
        >
          {paused ? "▶" : "⏸"}
        </button>
      </div>

      {/* Board */}
      <div style={{
        position: "relative",
        border: "2px solid #222",
        boxShadow: flash
          ? "0 0 40px #fff, 0 0 80px #ffe600"
          : "0 0 20px rgba(0,245,255,0.1)",
        transition: "box-shadow 0.1s",
        flex: "0 0 auto",
      }}>
        <canvas
          width={COLS * BK}
          height={ROWS * BK}
          style={{ display: "block" }}
          ref={canvas => {
            if (!canvas) return;
            const ctx = canvas.getContext("2d");
            ctx.fillStyle = "#050508";
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = "rgba(255,255,255,0.03)";
            ctx.lineWidth = 1;
            for (let r = 0; r <= ROWS; r++) {
              ctx.beginPath(); ctx.moveTo(0, r*BK); ctx.lineTo(COLS*BK, r*BK); ctx.stroke();
            }
            for (let c = 0; c <= COLS; c++) {
              ctx.beginPath(); ctx.moveTo(c*BK, 0); ctx.lineTo(c*BK, ROWS*BK); ctx.stroke();
            }
            for (let r = 0; r < ROWS; r++) {
              for (let c = 0; c < COLS; c++) {
                const cell = displayBoard[r][c];
                if (!cell) continue;
                const x = c*BK, y = r*BK;
                if (cell === "ghost") {
                  ctx.fillStyle = "rgba(255,255,255,0.07)";
                  ctx.fillRect(x+1, y+1, BK-2, BK-2);
                  ctx.strokeStyle = "rgba(255,255,255,0.18)";
                  ctx.lineWidth = 1;
                  ctx.strokeRect(x+1, y+1, BK-2, BK-2);
                } else {
                  ctx.fillStyle = cell;
                  ctx.fillRect(x+1, y+1, BK-2, BK-2);
                  ctx.fillStyle = "rgba(255,255,255,0.22)";
                  ctx.fillRect(x+1, y+1, BK-2, 3);
                  ctx.fillRect(x+1, y+1, 3, BK-2);
                  ctx.shadowColor = cell; ctx.shadowBlur = 6;
                  ctx.fillStyle = cell;
                  ctx.fillRect(x+2, y+2, BK-4, BK-4);
                  ctx.shadowBlur = 0;
                }
              }
            }
          }}
        />

        {(!started || gameOver) && (
          <div style={{
            position: "absolute", inset: 0, background: "rgba(5,5,8,0.9)",
            display: "flex", flexDirection: "column",
            alignItems: "center", justifyContent: "center", gap: 20,
          }}>
            <div style={{
              fontSize: gameOver ? 32 : 40, fontWeight: "bold",
              color: gameOver ? "#ff2a2a" : "#00f5ff",
              textShadow: `0 0 20px ${gameOver ? "#ff2a2a" : "#00f5ff"}`,
              letterSpacing: 6, textAlign: "center", whiteSpace: "pre-line",
            }}>{gameOver ? "GAME\nOVER" : "TETRIS"}</div>
            {gameOver && <div style={{ color: "#ffe600", fontSize: 16, letterSpacing: 3 }}>{score.toString().padStart(7,"0")}</div>}
            <button onClick={startGame} style={{
              background: "transparent", border: "2px solid #00f5ff",
              color: "#00f5ff", padding: "12px 32px", fontSize: 14,
              letterSpacing: 4, cursor: "pointer",
              textShadow: "0 0 10px #00f5ff",
              boxShadow: "0 0 20px rgba(0,245,255,0.3)", borderRadius: 4,
            }}>
              {gameOver ? "RETRY" : "START"}
            </button>
          </div>
        )}

        {paused && !gameOver && (
          <div style={{
            position: "absolute", inset: 0, background: "rgba(5,5,8,0.85)",
            display: "flex", alignItems: "center", justifyContent: "center",
          }}>
            <div style={{ color: "#c800ff", fontSize: 28, letterSpacing: 6, textShadow: "0 0 20px #c800ff" }}>PAUSE</div>
          </div>
        )}
      </div>

      {/* Touch controls */}
      <div style={{
        width: "100%", padding: "10px 16px 12px", boxSizing: "border-box",
        display: "flex", flexDirection: "column", gap: 10,
      }}>
        {/* Top row: rotate + hard drop */}
        <div style={{ display: "flex", gap: 10, justifyContent: "center" }}>
          <TouchBtn onPress={doRotate} color="#c800ff" style={{ width: 72, height: 52, fontSize: 24 }}>↻</TouchBtn>
          <TouchBtn onPress={hardDrop} color="#ffe600" style={{ flex: 1, height: 52, fontSize: 20, maxWidth: 160 }}>⬇ DROP</TouchBtn>
        </div>
        {/* Bottom row: left, down, right */}
        <div style={{ display: "flex", gap: 10, justifyContent: "center" }}>
          <TouchBtn onPress={moveLeft} color="#00f5ff" style={{ flex: 1, height: 56, fontSize: 28, maxWidth: 100 }}>◀</TouchBtn>
          <TouchBtn onPress={moveDown} color="#00ff41" style={{ flex: 1, height: 56, fontSize: 22, maxWidth: 100 }}>▼</TouchBtn>
          <TouchBtn onPress={moveRight} color="#00f5ff" style={{ flex: 1, height: 56, fontSize: 28, maxWidth: 100 }}>▶</TouchBtn>
        </div>
      </div>
    </div>
  );
}

function Stat({ label, value, color }) {
  return (
    <div style={{ textAlign: "center" }}>
      <div style={{ fontSize: 8, letterSpacing: 2, color: "#555", marginBottom: 2 }}>{label}</div>
      <div style={{ fontSize: 13, fontWeight: "bold", color, textShadow: `0 0 8px ${color}`, letterSpacing: 1 }}>{value}</div>
    </div>
  );
}
