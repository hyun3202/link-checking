<!DOCTYPE html>
<html lang="ko">
<head>
   <meta charset="UTF-8">
   <meta name="viewport" content="width=device-width, initial-scale=1.0">
   <title>AI Link Scanner</title>
   <script src="https://cdn.tailwindcss.com"></script>
   <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
   <style>
       @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@300;400;500;700&display=swap');
       body { font-family: 'Noto Sans KR', sans-serif; }
       .gradient-bg { background: linear-gradient(135deg, #1e3a8a 0%, #3b82f6 100%); }
   </style>
</head>
<body class="bg-gray-50 min-h-screen flex flex-col items-center py-10 px-4">


   <!-- Header -->
   <header class="text-center mb-10">
       <div class="inline-block p-4 rounded-full bg-blue-100 mb-4">
           <i class="fas fa-shield-alt text-4xl text-blue-600"></i>
       </div>
       <h1 class="text-3xl font-bold text-gray-800 mb-2">지능형 링크 검사기</h1>
       <p class="text-gray-500">CSV 데이터베이스와 패턴 분석 알고리즘을 결합한 하이브리드 탐지</p>
   </header>


   <!-- Main Container -->
   <main class="w-full max-w-2xl bg-white rounded-2xl shadow-xl overflow-hidden">


       <!-- 1. CSV Upload Section -->
       <div class="p-6 border-b border-gray-100 bg-gray-50">
           <h2 class="text-sm font-semibold text-gray-500 uppercase tracking-wider mb-3">
               <i class="fas fa-database mr-2"></i>데이터베이스 로드 (선택사항)
           </h2>
           <div class="flex items-center space-x-4">
               <label class="flex-1 flex flex-col items-center px-4 py-4 bg-white text-blue rounded-lg shadow-sm tracking-wide uppercase border border-blue-200 cursor-pointer hover:bg-blue-50 transition-colors">
                   <i class="fas fa-cloud-upload-alt text-2xl text-blue-500 mb-1"></i>
                   <span class="text-sm leading-normal text-gray-600" id="file-label">urls.csv 파일 선택 또는 드래그</span>
                   <input type='file' class="hidden" id="csvInput" accept=".csv" />
               </label>
           </div>
           <div id="db-status" class="mt-2 text-xs text-gray-400 text-right">
               현재 데이터: 0개 로드됨
           </div>
       </div>


       <!-- 2. URL Input Section -->
       <div class="p-8">
           <label class="block text-gray-700 text-sm font-bold mb-2" for="urlInput">
               검사할 URL 주소
           </label>
           <div class="relative">
               <div class="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
                   <i class="fas fa-link text-gray-400"></i>
               </div>
               <input type="text" id="urlInput"
                   class="w-full pl-10 pr-4 py-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-shadow"
                   placeholder="https://example.com/login...">
           </div>


           <button onclick="analyzeUrl()"
               class="w-full mt-4 gradient-bg text-white font-bold py-3 px-4 rounded-lg shadow-lg hover:shadow-xl transform hover:-translate-y-0.5 transition-all duration-200">
               검사 시작
           </button>
       </div>


       <!-- 3. Result Section (Hidden by default) -->
       <div id="resultArea" class="hidden bg-gray-100 border-t border-gray-200 p-8">
           <div id="resultCard" class="bg-white rounded-xl shadow-sm p-6 border-l-4">
               <div class="flex justify-between items-start mb-4">
                   <div>
                       <h3 class="text-xl font-bold" id="resultTitle">판정 결과</h3>
                       <p class="text-sm text-gray-500 mt-1" id="resultSubtitle">분석 완료</p>
                   </div>
                   <div class="text-center">
                       <div class="text-3xl font-bold" id="riskScore">0</div>
                       <div class="text-xs text-gray-400 uppercase">위험 점수</div>
                   </div>
               </div>


               <!-- Analysis Details -->
               <div class="space-y-2 mt-6">
                   <h4 class="text-sm font-semibold text-gray-700 border-b pb-2 mb-3">상세 분석 요인</h4>
                   <ul id="featureList" class="text-sm space-y-2">
                       <!-- Javascript will populate this -->
                   </ul>
               </div>
           </div>
       </div>


   </main>


   <footer class="mt-10 text-center text-gray-400 text-sm">
       <p>※ 이 도구는 참고용이며, 100% 정확성을 보장하지 않습니다.</p>
   </footer>


   <script>
       // --- 1. Global Variables ---
       let urlDatabase = {}; // CSV Data Storage


       // --- 2. CSV Parsing Logic ---
       document.getElementById('csvInput').addEventListener('change', function(e) {
           const file = e.target.files[0];
           if (!file) return;


           document.getElementById('file-label').innerText = file.name;


           const reader = new FileReader();
           reader.onload = function(e) {
               const text = e.target.result;
               const rows = text.split('\n');


               urlDatabase = {}; // Reset DB
               let count = 0;


               // Simple CSV Parser (Assuming Header exists: url, label)
               // Skip header (index 0) if necessary, simplified logic here
               rows.forEach((row, index) => {
                   if (index === 0) return; // Skip header assuming standard CSV


                   // Handle CSV splitting (basic comma split)
                   const cols = row.split(',');
                   if (cols.length >= 2) {
                       const url = cols[0].trim().toLowerCase(); // Normalize
                       const label = cols[1].trim();
                       if(url) {
                           urlDatabase[url] = label;
                           count++;
                       }
                   }
               });


               document.getElementById('db-status').innerText = `현재 데이터: ${count}개 로드됨 (CSV 기반)`;
               document.getElementById('db-status').classList.add('text-green-600');
               alert(`${count}개의 URL 데이터가 로드되었습니다.`);
           };
           reader.readAsText(file);
       });


       // --- 3. Analysis Logic (Converted from Python) ---


       function extractFeatures(urlStr) {
           let features = {
               details: [],
               score: 0
           };


           let urlObj;
           try {
               // Add protocol if missing for parsing
               if (!urlStr.match(/^https?:\/\//)) {
                   urlObj = new URL('http://' + urlStr);
               } else {
                   urlObj = new URL(urlStr);
               }
           } catch (e) {
               return null; // Invalid URL
           }


           const hostname = urlObj.hostname;
           const fullUrl = urlStr.toLowerCase();


           // 1. Has '@'
           if (fullUrl.includes('@')) {
               features.score += 3;
               features.details.push({ msg: "URL에 '@' 기호 포함 (난독화 의심)", risk: "high" });
           }


           // 2. Uses IP Address
           const ipRegex = /\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}/;
           if (ipRegex.test(hostname)) {
               features.score += 3;
               features.details.push({ msg: "도메인 대신 IP 주소 사용", risk: "high" });
           }


           // 3. HTTP usage (Not HTTPS)
           // Note: browser JS URL parsing might force protocol if not provided, check original string
           if (urlStr.startsWith('http://')) {
               features.score += 1;
               features.details.push({ msg: "보안되지 않은 HTTP 프로토콜 사용", risk: "medium" });
           }


           // 4. Suspicious Words
           const suspiciousWords = ["login", "verify", "secure", "account", "update", "admin", "bank", "signin", "bonus", "free"];
           const foundWords = suspiciousWords.filter(word => fullUrl.includes(word));
           if (foundWords.length > 0) {
               features.score += 2;
               features.details.push({ msg: `의심스러운 키워드 포함: ${foundWords.join(', ')}`, risk: "medium" });
           }


           // 5. Long URL
           if (fullUrl.length > 75) {
               features.score += 1;
               features.details.push({ msg: "URL 길이가 비정상적으로 긺 (>75자)", risk: "low" });
           }


           // 6. Many Subdomains
           const subdomainCount = (hostname.match(/\./g) || []).length;
           if (subdomainCount >= 3) {
               features.score += 2;
               features.details.push({ msg: "서브 도메인이 과도하게 많음", risk: "medium" });
           }


           // 7. Port usage
           if (urlObj.port) {
               features.score += 1;
               features.details.push({ msg: `비표준 포트 사용 (${urlObj.port})`, risk: "low" });
           }


           return features;
       }


       function analyzeUrl() {
           const urlInput = document.getElementById('urlInput').value.trim();
           const resultArea = document.getElementById('resultArea');
           const resultCard = document.getElementById('resultCard');
           const resultTitle = document.getElementById('resultTitle');
           const resultSubtitle = document.getElementById('resultSubtitle');
           const riskScoreDisplay = document.getElementById('riskScore');
           const featureList = document.getElementById('featureList');


           if (!urlInput) {
               alert("URL을 입력해주세요.");
               return;
           }


           // Reset UI
           resultArea.classList.remove('hidden');
           featureList.innerHTML = '';


           // --- Step 1: Check CSV Database ---
           const normalizedUrl = urlInput.toLowerCase();


           if (urlDatabase.hasOwnProperty(normalizedUrl)) {
               const label = urlDatabase[normalizedUrl];
               // CSV found
               if (label === "1") {
                   // Malicious in CSV
                   renderResult("danger", "악성 링크 (DB 탐지)", "업로드된 데이터베이스에 악성으로 등록된 URL입니다.", 100, []);
                   return;
               } else if (label === "0") {
                   // Safe in CSV
                   renderResult("safe", "정상 링크 (DB 확인)", "업로드된 데이터베이스에 정상으로 등록된 URL입니다.", 0, []);
                   return;
               }
           }


           // --- Step 2: Heuristic Analysis ---
           const features = extractFeatures(urlInput);


           if (!features) {
               renderResult("warning", "유효하지 않은 URL", "올바른 URL 형식이 아닙니다.", 0, []);
               return;
           }


           let status = "safe";
           let statusText = "정상 링크 가능성 높음";
           let desc = "특이한 보안 위협 요소가 발견되지 않았습니다.";


           if (features.score >= 2) {
               status = "danger";
               statusText = "악성 링크 의심";
               desc = "다수의 보안 위협 패턴이 감지되었습니다. 접속에 주의하세요.";
           }


           // Render features details
           let detailHtmlItems = features.details.map(item => {
               const colorClass = item.risk === 'high' ? 'text-red-500 font-bold' : (item.risk === 'medium' ? 'text-orange-500' : 'text-gray-600');
               const icon = item.risk === 'high' ? '<i class="fas fa-exclamation-circle"></i>' : '<i class="fas fa-check-circle"></i>';
               return `<li class="flex items-center space-x-2 ${colorClass}">${icon} <span>${item.msg}</span></li>`;
           });


           if(detailHtmlItems.length === 0) {
               detailHtmlItems.push('<li class="flex items-center space-x-2 text-green-600"><i class="fas fa-shield-alt"></i> <span>위협 패턴이 발견되지 않음</span></li>');
           }


           renderResult(status, statusText, desc, features.score, detailHtmlItems);
       }


       function renderResult(type, title, subtitle, score, listItems) {
           const card = document.getElementById('resultCard');
           const titleEl = document.getElementById('resultTitle');
           const subtitleEl = document.getElementById('resultSubtitle');
           const scoreEl = document.getElementById('riskScore');
           const listEl = document.getElementById('featureList');


           // Reset classes
           card.classList.remove('border-green-500', 'border-red-500', 'border-yellow-500');
           titleEl.classList.remove('text-green-600', 'text-red-600', 'text-yellow-600');
           scoreEl.classList.remove('text-green-600', 'text-red-600', 'text-yellow-600');


           if (type === 'safe') {
               card.classList.add('border-green-500');
               titleEl.classList.add('text-green-600');
               scoreEl.classList.add('text-green-600');
           } else if (type === 'danger') {
               card.classList.add('border-red-500');
               titleEl.classList.add('text-red-600');
               scoreEl.classList.add('text-red-600');
           } else {
               card.classList.add('border-yellow-500');
               titleEl.classList.add('text-yellow-600');
               scoreEl.classList.add('text-yellow-600');
           }


           titleEl.innerText = title;
           subtitleEl.innerText = subtitle;
           scoreEl.innerText = score;


           if (listItems.length > 0) {
               listEl.innerHTML = listItems.join('');
           } else {
               listEl.innerHTML = '';
           }
       }


   </script>
</body>
</html>

