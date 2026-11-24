# multilingual-voice-recorder

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Multilingual Voice Recorder</title>

<style>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  padding: 20px;
  background: #101a32;
  color: white;
  font-family: Arial, sans-serif;
}

.container {
  max-width: 600px;
  margin: auto;
}

h2 {
  color: #00d9ff;
  text-align: center;
}

.panel {
  background: #1c2b49;
  padding: 18px;
  border-radius: 12px;
  margin-top: 16px;
}

select, button {
  width: 100%;
  padding: 13px;
  margin-top: 10px;
  border: none;
  border-radius: 8px;
  font-size: 16px;
}

select {
  background: white;
  color: #101a32;
}

button {
  background: #00c9e8;
  color: #101a32;
  font-weight: bold;
}

button:disabled {
  opacity: 0.4;
}

.stop {
  background: #ff5364;
  color: white;
}

textarea {
  width: 100%;
  min-height: 200px;
  padding: 12px;
  border-radius: 8px;
  font-size: 16px;
  resize: vertical;
}

.status {
  color: #00e5ff;
  line-height: 1.6;
  overflow-wrap: anywhere;
}

audio {
  width: 100%;
  margin-top: 12px;
}

.download {
  display: block;
  color: #00e5ff;
  margin-top: 15px;
  overflow-wrap: anywhere;
}
</style>
</head>

<body>

<div class="container">

<h2>🎙️ Multilingual Voice Recorder</h2>

<div class="panel">

  <h3>1. Select Language</h3>

  <select id="language">
    <option value="en-IN">English</option>
    <option value="ta-IN">Tamil</option>
    <option value="en-IN">Tanglish (Experimental)</option>
  </select>

  <button id="startBtn">▶ Start Recording</button>

  <button id="stopBtn" class="stop" disabled>
    ■ Stop Recording
  </button>

  <p class="status" id="audioStatus">
    Audio: Ready
  </p>

  <p class="status" id="speechStatus">
    Speech: Ready
  </p>

</div>

<div class="panel">

  <h3>2. Live Transcript</h3>

  <textarea id="transcript"
    placeholder="Your spoken words will appear here..."
    readonly></textarea>

  <button id="textDownloadBtn">
    Download Transcript (TXT)
  </button>

</div>

<div class="panel">

  <h3>3. Audio Recording</h3>

  <audio id="audioPlayer" controls></audio>

  <a id="audioDownload" class="download"
     style="display:none">
    Download Audio
  </a>

</div>

</div>

<script>

const SR =
  window.SpeechRecognition ||
  window.webkitSpeechRecognition;

const startBtn = document.getElementById("startBtn");
const stopBtn = document.getElementById("stopBtn");

const transcript = document.getElementById("transcript");
const audioStatus = document.getElementById("audioStatus");
const speechStatus = document.getElementById("speechStatus");

const audioPlayer = document.getElementById("audioPlayer");
const audioDownload = document.getElementById("audioDownload");

let audioStream = null;
let mediaRecorder = null;
let audioChunks = [];

let recognition = null;
let restartTimer = null;

let isRecording = false;
let isRecognizing = false;
let shouldRecognize = false;
let recognitionBlocked = false;

let savedFinal = "";
let currentFinal = "";

let audioURL = null;


// Display transcript
function updateTranscript(interim = "") {

  transcript.value =
    savedFinal + currentFinal + interim;

}


// Start one speech-recognition session
function startRecognition() {

  if (!isRecording || !shouldRecognize) return;
  if (!SR) {
    speechStatus.innerText =
      "Speech: Not supported by this browser.";
    return;
  }

  if (isRecognizing) return;

  const rec = new SR();
  recognition = rec;

  rec.lang = document.getElementById("language").value;
  rec.continuous = true;
  rec.interimResults = true;
  rec.maxAlternatives = 1;

  currentFinal = "";

  rec.onstart = function() {

    if (recognition !== rec) return;

    isRecognizing = true;

    speechStatus.innerText =
      "Speech: Listening live...";

  };


  rec.onresult = function(event) {

    if (recognition !== rec) return;

    let finalText = "";
    let interimText = "";

    for (let i = 0; i < event.results.length; i++) {

      const result = event.results[i];
      const spokenText = result[0].transcript;

      if (result.isFinal) {
        finalText += spokenText + " ";
      } else {
        interimText += spokenText;
      }

    }

    currentFinal = finalText;

    updateTranscript(interimText);

    speechStatus.innerText =
      "Speech: Live transcript received";

  };


  rec.onerror = function(event) {

    if (recognition !== rec) return;

    speechStatus.innerText =
      "Speech error: " + event.error;

    if (
      event.error === "not-allowed" ||
      event.error === "service-not-allowed"
    ) {

      recognitionBlocked = true;
      shouldRecognize = false;

    }

  };


  rec.onend = function() {

    if (recognition !== rec) return;

    isRecognizing = false;

    // Save final words from this session
    savedFinal += currentFinal;
    currentFinal = "";

    updateTranscript();

    if (isRecording && shouldRecognize &&
        !recognitionBlocked) {

      speechStatus.innerText =
        "Speech: Restarting recognition...";

      clearTimeout(restartTimer);

      restartTimer = setTimeout(function() {

        if (isRecording && shouldRecognize) {
          startRecognition();
        }

      }, 700);

    } else if (!isRecording) {

      transcript.readOnly = false;

      speechStatus.innerText =
        "Speech: Finished";

    }

  };


  try {

    rec.start();

  } catch (error) {

    isRecognizing = false;

    speechStatus.innerText =
      "Speech start error: " + error.message;

  }

}


// Start audio and speech together
async function startRecording() {

  if (isRecording) return;

  if (!navigator.mediaDevices ||
      !navigator.mediaDevices.getUserMedia ||
      !window.MediaRecorder) {

    audioStatus.innerText =
      "Audio: Not supported in this browser.";
    return;

  }

  if (!SR) {

    speechStatus.innerText =
      "Speech: Not supported in this browser.";

    return;

  }

  startBtn.disabled = true;
  stopBtn.disabled = false;

  audioChunks = [];
  savedFinal = "";
  currentFinal = "";

  transcript.value = "";
  transcript.readOnly = true;

  recognitionBlocked = false;

  try {

    audioStatus.innerText =
      "Audio: Requesting microphone...";

    audioStream =
      await navigator.mediaDevices.getUserMedia({
        audio: true
      });

    const mimeType =
      MediaRecorder.isTypeSupported("audio/webm;codecs=opus")
        ? "audio/webm;codecs=opus"
        : "";

    mediaRecorder = mimeType
      ? new MediaRecorder(audioStream, { mimeType })
      : new MediaRecorder(audioStream);

    mediaRecorder.ondataavailable = function(event) {

      if (event.data && event.data.size > 0) {
        audioChunks.push(event.data);
      }

    };


    mediaRecorder.onstop = function() {

      const blob = new Blob(audioChunks, {
        type: mediaRecorder.mimeType || "audio/webm"
      });

      if (audioURL) {
        URL.revokeObjectURL(audioURL);
      }

      audioURL = URL.createObjectURL(blob);

      audioPlayer.src = audioURL;

      audioDownload.href = audioURL;
      audioDownload.download = "voice-recording.webm";
      audioDownload.style.display = "block";
      audioDownload.innerText = "⬇ Download Audio";

      audioStatus.innerText =
        "Audio: Recording saved";

      if (audioStream) {
        audioStream.getTracks().forEach(track => track.stop());
        audioStream = null;
      }

    };


    mediaRecorder.start();

    isRecording = true;
    shouldRecognize = true;

    audioStatus.innerText =
      "Audio: Recording...";

    startRecognition();

  } catch (error) {

    isRecording = false;
    shouldRecognize = false;

    startBtn.disabled = false;
    stopBtn.disabled = true;

    audioStatus.innerText =
      "Audio error: " + error.message;

    speechStatus.innerText =
      "Speech: Not started";

    if (audioStream) {
      audioStream.getTracks().forEach(track => track.stop());
      audioStream = null;
    }

  }

}


// Stop recording
function stopRecording() {

  if (!isRecording) return;

  isRecording = false;
  shouldRecognize = false;

  clearTimeout(restartTimer);

  audioStatus.innerText =
    "Audio: Stopping...";

  speechStatus.innerText =
    "Speech: Finishing...";

  if (recognition && isRecognizing) {
    recognition.stop();
  } else {
    transcript.readOnly = false;
    speechStatus.innerText = "Speech: Finished";
  }

  if (mediaRecorder &&
      mediaRecorder.state !== "inactive") {

    mediaRecorder.stop();

  }

  startBtn.disabled = false;
  stopBtn.disabled = true;

}


// Download transcript
function downloadText() {

  const text = transcript.value.trim();

  if (!text) {

    alert("No transcript available.");
    return;

  }

  const blob = new Blob(
    ["\uFEFF" + text],
    { type: "text/plain;charset=utf-8" }
  );

  const url = URL.createObjectURL(blob);

  const link = document.createElement("a");

  link.href = url;
  link.download = "voice-transcript.txt";

  link.click();

  URL.revokeObjectURL(url);

}


// Button actions
startBtn.addEventListener("click", startRecording);
stopBtn.addEventListener("click", stopRecording);

document.getElementById("textDownloadBtn")
  .addEventListener("click", downloadText);

</script>

</body>
</html>
