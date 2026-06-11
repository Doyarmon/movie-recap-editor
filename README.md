
<!DOCTYPE html>
<html lang="my">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HTML5 Movie Recap Editor</title>
    <style>
        :root {
            --primary: #ff4757;
            --bg: #1e1e24;
            --card: #2f3542;
            --text: #ffffff;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: var(--bg);
            color: var(--text);
            margin: 0;
            padding: 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
        }
        .container {
            max-width: 800px;
            width: 100%;
            background: var(--card);
            padding: 30px;
            border-radius: 12px;
            box-shadow: 0 8px 24px rgba(0,0,0,0.3);
        }
        h2 { text-align: center; color: var(--primary); margin-top: 0; }
        .upload-section {
            border: 2px dashed #57606f;
            padding: 30px;
            text-align: center;
            border-radius: 8px;
            cursor: pointer;
            margin-bottom: 20px;
        }
        .upload-section:hover { border-color: var(--primary); }
        video { width: 100%; border-radius: 8px; margin-top: 15px; background: #000; }
        .time-controls {
            background: #20242c;
            padding: 15px;
            border-radius: 8px;
            margin-top: 15px;
        }
        .segment-row {
            display: flex;
            gap: 10px;
            align-items: center;
            margin-bottom: 10px;
        }
        input[type="number"] {
            width: 80px;
            padding: 8px;
            border-radius: 4px;
            border: 1px solid #57606f;
            background: #1e1e24;
            color: white;
        }
        button {
            padding: 10px 20px;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-weight: bold;
            transition: 0.2s;
        }
        .btn-add { background: #2ed573; color: white; }
        .btn-remove { background: #ff4757; color: white; padding: 8px 12px; }
        .btn-process {
            background: var(--primary);
            color: white;
            width: 100%;
            font-size: 16px;
            padding: 15px;
            margin-top: 20px;
        }
        button:hover { opacity: 0.9; }
        #status {
            margin-top: 15px;
            text-align: center;
            font-weight: bold;
            color: #ffa502;
        }
    </style>
    <script src="https://unpkg.com/@ffmpeg/ffmpeg@0.11.6/dist/ffmpeg.min.js"></script>
</head>
<body>

<div class="container">
    <h2>🎬 Movie Recap Video Editor</h2>
    
    <div class="upload-section" onclick="document.getElementById('videoInput').click()">
        <p>📁 ဗီဒီယိုဖိုင် ရွေးချယ်ရန် သို့မဟုတ် Drag & Drop လုပ်ရန် နှိပ်ပါ</p>
        <input type="file" id="videoInput" accept="video/mp4" style="display:none">
    </div>

    <video id="mainVideo" controls style="display:none;"></video>

    <div id="editorSection" style="display:none;">
        <div class="time-controls">
            <h3>📌 ဖြတ်ထုတ်မည့် အရေးကြီးသော အပိုင်းများ (စက္ကန့်အလိုက်)</h3>
            <div id="segmentsContainer">
                <div class="segment-row">
                    <label>စမည့်အချိန်:</label>
                    <input type="number" class="start-time" value="0" min="0">
                    <label>ဆုံးမည့်အချိန်:</label>
                    <input type="number" class="end-time" value="10" min="0">
                </div>
            </div>
            <button class="btn-add" onclick="addSegmentRow()">➕ အပိုင်းအသစ် ထပ်တိုးရန်</button>
        </div>

        <button class="btn-process" onclick="processRecapVideo()">🪄 Recap ဗီဒီယို စတင်ထုတ်လုပ်မည်</button>
    </div>

    <div id="status"></div>
    <div id="resultSection" style="margin-top:20px; display:none; text-align:center;">
        <h3>✨ ဖန်တီးပြီးသား Movie Recap ဗီဒီယို</h3>
        <video id="outputVideo" controls></video>
        <br><br>
        <a id="downloadBtn" class="btn-add" style="text-decoration:none; padding:12px 25px; display:inline-block;" download="movie_recap.mp4">📥 Recap ဗီဒီယိုကို ကွန်ပျူတာထဲသိမ်းရန်</a>
    </div>
</div>

<script>
    const { createFFmpeg, fetchFile } = FFmpeg;
    const ffmpeg = createFFmpeg({ log: true });
    
    let videoFile = null;
    const videoInput = document.getElementById('videoInput');
    const mainVideo = document.getElementById('mainVideo');
    const editorSection = document.getElementById('editorSection');
    const segmentsContainer = document.getElementById('segmentsContainer');
    const statusDiv = document.getElementById('status');

    // ၁။ ဗီဒီယိုတင်လိုက်သည့်အခါ အလုပ်လုပ်ပုံ
    videoInput.addEventListener('change', function(e) {
        videoFile = e.target.files[0];
        if (videoFile) {
            const url = URL.createObjectURL(videoFile);
            mainVideo.src = url;
            mainVideo.style.display = 'block';
            editorSection.style.display = 'block';
            statusDiv.innerText = "ဗီဒီယို တင်သွင်းပြီးပါပြီ။ ဖြတ်တောက်မည့် စက္ကန့်များကို သတ်မှတ်ပါ။";
        }
    });

    // ၂။ အပိုင်းအသစ် (Row) ထပ်တိုးပေးသည့် Function
    function addSegmentRow() {
        const row = document.createElement('div');
        row.className = 'segment-row';
        row.innerHTML = `
            <label>Def စမည့်အချိန်:</label>
            <input type="number" class="start-time" value="0" min="0">
            <label>ဆုံးမည့်အချိန်:</label>
            <input type="number" class="end-time" value="10" min="0">
            <button class="btn-remove" onclick="this.parentElement.remove()">❌</button>
        `;
        segmentsContainer.appendChild(row);
    }

    // ၃။ FFmpeg ဖြင့် ဗီဒီယိုကို Browser ပေါ်တွင်တင် ဖြတ်တောက်ပေါင်းစပ်ခြင်း
    async function processRecapVideo() {
        if (!videoFile) return alert("ဗီဒီယို အရင်တင်ပေးပါဦး။");
        
        statusDiv.innerText = "FFmpeg Core ကို အသက်သွင်းနေပါသည်... ခဏစောင့်ပါ။";
        if (!ffmpeg.isLoaded()) {
            await ffmpeg.load();
        }

        statusDiv.innerText = "ဗီဒီယိုကို ဖတ်ရှုနေပါသည်...";
        ffmpeg.FS('writeFile', 'input.mp4', await fetchFile(videoFile));

        const startInputs = document.querySelectorAll('.start-time');
        const endInputs = document.querySelectorAll('.end-time');
        
        let concatListContent = '';
        statusDiv.innerText = "ဗီဒီယို အပိုင်းအစများကို ဖြတ်တောက်နေပါသည်...";

        // အပိုင်းတစ်ခုချင်းစီကို လိုက်ဖြတ်ခြင်း
        for (let i = 0; i < startInputs.length; i++) {
            const start = startInputs[i].value;
            const duration = endInputs[i].value - start;
            const partName = `part${i}.mp4`;

            if (duration <= 0) continue;

            // FFmpeg Command ဖြင့် ဖြတ်ထုတ်ခြင်း
            await ffmpeg.run('-ss', `${start}`, '-i', 'input.mp4', '-t', `${duration}`, '-c:v', 'copy', '-c:a', 'copy', partName);
            concatListContent += `file ${partName}\n`;
        }

        // ဖြတ်ထားတဲ့ အပိုင်းတွေကို ပြန်ပေါင်းဖို့ စာရင်းပြုစုခြင်း
        ffmpeg.FS('writeFile', 'list.txt', new TextEncoder().encode(concatListContent));

        statusDiv.innerText = "အပိုင်းအားလုံးကို Recap ဗီဒီယိုတစ်ခုတည်းအဖြစ် ပေါင်းစပ်နေပါသည်...";
        // ဗီဒီယိုအားလုံးကို ပြန်ပေါင်းခြင်း
        await ffmpeg.run('-f', 'concat', '-safe', '0', '-i', 'list.txt', '-c', 'copy', 'output.mp4');

        statusDiv.innerText = "ပြီးပါပြီ။ ဗီဒီယိုဖိုင် ထုတ်ယူနေပါသည်...";
        const data = ffmpeg.FS('readFile', 'output.mp4');

        // ရလာတဲ့ ဗီဒီယိုကို Browser ပေါ်တွင် ပြသခြင်း
        const resultVideoBlob = new Blob([data.buffer], { type: 'video/mp4' });
        const resultUrl = URL.createObjectURL(resultVideoBlob);
        
        document.getElementById('outputVideo').src = resultUrl;
        document.getElementById('downloadBtn').href = resultUrl;
        document.getElementById('resultSection').style.display = 'block';
        statusDiv.innerText = "✨ Movie Recap ဗီဒီယို အောင်မြင်စွာ ပြုလုပ်ပြီးပါပြီ!";
    }
</script>

</body>
</html>
