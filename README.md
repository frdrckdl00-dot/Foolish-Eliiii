<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Lyrics Karaoke</title>

  <!-- Poppins font -->
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@700&display=swap" rel="stylesheet">

  <style>
    :root{
      --bg: #000;
      --text: #fff;
      --fade-duration: 500ms; /* sync with JS fade timing */
    }

    html,body{
      height:100%;
      margin:0;
      font-family: "Poppins", system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
      background: var(--bg);
      color: var(--text);
    }

    /* center everything */
    .wrap {
      height:100vh;
      display:flex;
      align-items:center;
      justify-content:center;
      flex-direction:column;
      text-align:center;
      padding: 20px;
      box-sizing:border-box;
    }

    /* large lyric area */
    .lyric-box{
      width:100%;
      max-width:960px;
      min-height:160px;
      display:flex;
      align-items:center;
      justify-content:center;
      position:relative;
      overflow:hidden;
    }

    .lyric {
      font-weight:700;
      font-size:2.4rem;
      line-height:1.2;
      opacity:0;
      transform: translateY(8px);
      transition:
        opacity var(--fade-duration) ease,
        transform var(--fade-duration) ease;
      pointer-events:none;
      white-space:pre-wrap;
      padding: 0 20px;
    }

    .lyric.visible{
      opacity:1;
      transform: translateY(0);
    }

    /* small play button */
    .play-btn{
      margin-top:30px;
      background:transparent;
      border: 2px solid var(--text);
      color:var(--text);
      padding: 12px 22px;
      border-radius:32px;
      cursor:pointer;
      font-weight:700;
      font-family: inherit;
      font-size:1rem;
      transition: all 300ms ease;
      display:inline-flex;
      align-items:center;
      gap:10px;
    }

    .play-btn:hover{ transform: scale(1.03); }

    /* hidden after click */
    .play-btn.hidden{
      opacity:0;
      transform: scale(.98);
      pointer-events:none;
      transition: opacity 400ms ease, transform 400ms ease;
    }

    /* final message style */
    .final-msg{
      margin-top:28px;
      font-weight:700;
      font-size:1.1rem;
      opacity:0;
      transform: translateY(6px);
      transition: opacity 400ms ease, transform 400ms ease;
    }

    .final-msg.show{
      opacity:1;
      transform: translateY(0);
    }

    /* make it responsive */
    @media (max-width:480px){
      .lyric { font-size:1.6rem; }
      .play-btn { padding:10px 16px; }
    }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="lyric-box" aria-live="polite" aria-atomic="true">
      <div id="lyric" class="lyric"></div>
    </div>

    <button id="playBtn" class="play-btn" aria-label="Play song">
      ▶ Play
    </button>

    <div id="finalMsg" class="final-msg">performative pala ha bleeeeeee</div>
  </div>

  <!-- Hidden audio element. Put your file named exactly "your-audio-file.mp3" in same folder. -->
  <audio id="audio" src="your-audio-file.mp3" preload="auto"></audio>

  <script>
    /*
      Lyrics sequence:
      Each item: { text: "...", dur: seconds }
      Blank entries are represented by an empty string for 'text'
    */
    const lyrics = [
      { text: "You know how to keep me waitin'", dur: 0.02 * 1000 },      // (0:02) -> 2 seconds (given as 0:02)
      { text: "I know how to act like I'm fine", dur: 0.03 * 1000 },     // 3s
      { text: "Don't know what to call this situation", dur: 0.04 * 1000 }, // 4s
      { text: "But I know I can't call you mine", dur: 1.5 * 1000 },     // 1.5s
      { text: "And it's delicate, but I will do my best to seem bulletproof", dur: 5.5 * 1000 }, //5.5s
      { text: "", dur: 3 * 1000 },                                       // blank 3s
      { text: "'Cause when my head is on your shoulder", dur: 3 * 1000 }, //3s
      { text: "It starts thinkin' you'll come around", dur: 2 * 1000 },    //2s
      { text: "And maybe, someday, when we're older", dur: 2.7 * 1000 },  //2.7s
      { text: "This is something we'll laugh about", dur: 2 * 1000 },     //2s (treated as 2s)
      { text: "Over coffee every mornin' while you're watching the news", dur: 6 * 1000 }, //6s
      { text: "", dur: 2 * 1000 },                                       // 2s blank
      { text: "But then the voices say, \"You are not the exception", dur: 6 * 1000 }, //6s
      { text: "You will never learn your lesson\"", dur: 5 * 1000 },     //5s
      { text: "Foolish one", dur: 1.5 * 1000 },                          //1.5s
      { text: "Stop checkin' your mailbox for confessions of love", dur: 5 * 1000 }, //5s
      { text: "That ain't never gonna come", dur: 3 * 1000 },            //3s
      { text: "You will take the long way, you will take the long way down", dur: 6 * 1000 } //6s
    ];

    // NOTE: The user provided some timestamps in a short form (e.g. 0:02). I interpreted
    // them as whole seconds (2s, 3s, etc.). Entries that included decimals (like 1.5, 2.7, 5.5)
    // were used exactly. If you'd like different timing, edit the dur values above.

    const lyricEl = document.getElementById('lyric');
    const playBtn = document.getElementById('playBtn');
    const audio = document.getElementById('audio');
    const finalMsg = document.getElementById('finalMsg');

    // fade duration must match CSS -- used when waiting for fade-out to finish
    const FADE_MS = 500; // matches --fade-duration

    let seqAbort = false;

    async function playSequence(){
      seqAbort = false;
      for (let i = 0; i < lyrics.length; i++){
        if (seqAbort) break;
        const item = lyrics[i];

        // set text (could be blank)
        lyricEl.textContent = item.text;
        // small delay to ensure DOM updated before showing
        await tick();

        // show (fade in)
        lyricEl.classList.add('visible');

        // wait for the lyric's duration (display duration)
        await wait(item.dur);

        // fade out
        lyricEl.classList.remove('visible');

        // wait for fade-out to complete before next lyric
        await wait(FADE_MS);
      }

      if (!seqAbort){
        // show final message
        finalMsg.classList.add('show');
      }
    }

    // helper: wait ms
    function wait(ms){ return new Promise(res => setTimeout(res, ms)); }
    // next microtask
    function tick(){ return new Promise(res => setTimeout(res, 20)); }

    playBtn.addEventListener('click', async function onPlay(){
      // hide the play button
      playBtn.classList.add('hidden');
      playBtn.setAttribute('aria-hidden','true');

      // play audio (attempt)
      try {
        await audio.play();
      } catch (e) {
        console.warn("Audio play failed (browser autoplay policy). Click to start audio manually.", e);
      }

      // start lyric sequence
      playSequence();
    });

    // if user pauses audio or seeks, you might want to stop the sequence -- here's a simple approach:
    audio.addEventListener('pause', () => {
      // do not abort sequence on pause so lyrics can continue in this implementation.
      // If you'd rather stop, set seqAbort = true;
    });

    // Optional: if user reloads or navigates away stop timers (cleanup)
    window.addEventListener('beforeunload', () => { seqAbort = true; });

    // Accessibility: allow Space/Enter to trigger the play button
    playBtn.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); playBtn.click(); }
    });
  </script>
</body>
</html>
