<!DOCTYPE html>
<html lang="ko">
<head>
   <meta charset="UTF-8">
   <meta name="viewport" content="width=device-width, initial-scale=1.0">
   <title>AI Link Scanner - Pro</title>
   <script src="https://cdn.tailwindcss.com"></script>
   <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
   <style>
       @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@300;400;500;700&display=swap');
       body { font-family: 'Noto Sans KR', sans-serif; }
       .gradient-bg { background: linear-gradient(135deg, #1e3a8a 0%, #3b82f6 100%); }
       /* 로딩 애니메이션 */
       .status-loading { animation: pulse 2s infinite; }
       @keyframes pulse { 0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } }
   </style>
</head>
<body class="bg-gray-50 min-h-screen flex flex-col items-center py-10 px-4">

   <header class="text-center mb-10">
       <div class="inline-block p-4 rounded-full bg-blue-100 mb-4">
           <i class="fas fa-shield-alt text-4xl text-blue-600"></i>
       </div>
       <h1 class="text-3xl font-bold text-gray-800 mb-2">지능형 링크 검사기</h1>
       <p class="text-gray-500">27,000+ 블랙리스트 DB 및 실시간 패턴 분석</p>
   </header>

   <main class="w-full max-w-2xl bg-white rounded-2xl shadow-xl overflow-hidden">
       
       <div class="p-4 border-b border-gray-100 bg-gray-50 flex justify-between items-center px-8">
           <div class="flex items-center space-x-2">
               <div id="status-dot" class="w-3 h-3 rounded-full bg-yellow-400"></div>
               <span id="db-status" class="text-sm font-medium text-gray-600">데이터베이스 동기화 중...</span>
           </div>
           <div class="text-[10px] text-gray-400 font-mono" id="db-count">READY</div>
       </div>

       <div class="p-8">
           <label class="block text-gray-700 text-sm font-bold mb-2" for="urlInput">
               검사할 URL 주소
           </label>
           <div class="relative">
               <div class="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
                   <i class="fas fa-link text-gray-400"></i>
               </div>
               <input type="text" id="urlInput"
                   class="w-full pl-10 pr-4 py-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 transition-all"
                   placeholder="검사할 주소를 입력하거나 붙여넣으세요.">
           </div>

           <button onclick="analyzeUrl()"
               class="w-full mt-4 gradient-bg text-white font-bold py-4 px-4 rounded-lg shadow-lg hover:shadow-xl transform hover:-translate-y-0.5 transition-all duration-200">
               실시간 보안 검사 시작
           </button>
       </div>

       <div id="resultArea" class="hidden bg-gray-100 border-t border-gray-200 p-8">
           <div id="resultCard" class="bg-white rounded-xl shadow-sm p-6 border-l-8">
               <div class="flex justify-between items-start mb-6">
                   <div>
                       <h3 class="text-2xl font-bold" id="resultTitle">판정 결과</h3>
                       <p class="text-sm text-gray-500 mt-1" id="resultSubtitle">분석 완료</p>
                   </div>
                   <div class="text-center bg-gray-50 p-3 rounded-lg min-w-[80px]">
                       <div class="text-3xl font-bold" id="riskScore">0</div>
                       <div class="text-[10px] text-gray-400 uppercase tracking-tighter">위험 지수</div>
                   </div>
               </div>

               <div class="space-y-4">
                   <h4 class="text-xs font-bold text-gray-400 uppercase tracking-widest border-b pb-2">상세 분석 리포트</h4>
                   <ul id="featureList" class="space-y-3 text-sm">
                       </ul>
               </div>
           </div>
       </div>
   </main>

   <footer class="mt-10 text-center text-gray-400 text-xs space-y-2">
       <p><i class="fas fa-info-circle mr-1"></i> 본 도구는 데이터베이스와 알고리즘 기반의 참고용 도구입니다.</p>
       <p>© 2025 AI Link Scanner • Database: v1.0.27k</p>
   </footer>

   <script>
       // --- 1. 전역 변수 ---
       let urlDatabase = {};

       // --- 2. 페이지 로드 시 DB 자동 로드 (사용자 JSON 형식 최적화) ---
       window.addEventListener('DOMContentLoaded', () => {
           const statusText = document.getElementById('db-status');
           const statusDot = document.getElementById('status-dot');
           const dbCount = document.getElementById('db-count');

           // database.json 파일 읽기
           fetch('database.json')
               .then(response => {
                   if (!response.ok) throw new Error('파일을 찾을 수 없습니다.');
                   return response.json();
               })
               .then(data => {
                   // 배열 데이터를 빠른 검색을 위해 객체(Map)로 변환
                   // 구조: { "url": "...", "label": 1 }
                   data.forEach(item => {
                       if(item.url) {
                           // URL을 키로 사용 (소문자 정규화)
                           urlDatabase[item.url.trim().toLowerCase()] = String(item.label);
                       }
                   });

                   const count = Object.keys(urlDatabase).length;
                   statusText.innerText = `데이터베이스 최적화 완료`;
                   dbCount.innerText = `${count.toLocaleString()} URLS`;
                   statusDot.className = "w-3 h-3 rounded-full bg-green-500";
               })
               .catch(err => {
                   console.error("DB 로드 실패:", err);
                   statusText.innerText = "로컬 DB 로드 실패 (파일 확인 필요)";
                   statusDot.className = "w-3 h-3 rounded-full bg-red-500";
               });
       });

       // --- 3. 패턴 분석 엔진 (Heuristic) ---
       function extractFeatures(urlStr) {
           let features = { details: [], score: 0 };
           let urlObj;
           
           try {
               urlObj = new URL(urlStr.match(/^https?:\/\//) ? urlStr : 'http://' + urlStr);
           } catch (e) { return null; }

           const hostname = urlObj.hostname;
           const fullUrl = urlStr.toLowerCase();

           // 위험 패턴 체크
           if (fullUrl.includes('@')) { 
               features.score += 40; 
               features.details.push({ msg: "사용자 정보 탈취 의심 문구(@) 감지", risk: "high" }); 
           }
           if (/\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}/.test(hostname)) { 
               features.score += 50; 
               features.details.push({ msg: "비표준 IP 주소 기반 도메인 사용", risk: "high" }); 
           }
           if (urlStr.startsWith('http://')) { 
               features.score += 15; 
               features.details.push({ msg: "데이터 암호화 미적용 (HTTP)", risk: "medium" }); 
           }
           
           const suspiciousWords = ["login", "verify", "secure", "account", "bank", "update", "signin", "free", "bonus"];
           suspiciousWords.forEach(word => {
               if (fullUrl.includes(word)) { 
                   features.score += 20; 
                   features.details.push({ msg: `피싱 의심 키워드 발견: ${word}`, risk: "medium" }); 
               }
           });

           if (fullUrl.length > 80) { 
               features.score += 10; 
               features.details.push({ msg: "비정상적인 URL 길이 (난독화 의심)", risk: "low" }); 
           }

           return features;
       }

       // --- 4. 메인 검사 실행 ---
       function analyzeUrl() {
           const input = document.getElementById('urlInput').value.trim();
           if (!input) return alert("검사할 URL을 입력해주세요.");

           const resultArea = document.getElementById('resultArea');
           const normalizedUrl = input.toLowerCase();
           
           resultArea.classList.remove('hidden');

           // [Step 1] DB 블랙리스트 대조
           if (urlDatabase.hasOwnProperty(normalizedUrl)) {
               const label = urlDatabase[normalizedUrl];
               const isMalicious = (label === "1");
               
               renderResult(
                   isMalicious ? "danger" : "safe",
                   isMalicious ? "위험 사이트 판정" : "안전한 사이트",
                   isMalicious ? "2.7만개의 블랙리스트 DB와 정확히 일치합니다." : "화이트리스트 DB에 등록된 검증된 주소입니다.",
                   isMalicious ? 100 : 0,
                   isMalicious ? ['<li class="text-red-600 font-bold"><i class="fas fa-skull-crossbones mr-2"></i>악성 데이터베이스(DB) 검색 일치</li>'] 
                               : ['<li class="text-green-600 font-bold"><i class="fas fa-check-circle mr-2"></i>공식 데이터베이스 검증 완료</li>']
               );
               return;
           }

           // [Step 2] 패턴 분석 엔진 가동
           const features = extractFeatures(input);
           if (!features) {
               renderResult("warning", "형식 오류", "분석할 수 없는 URL 형식입니다.", 0, []);
               return;
           }

           let status = "safe";
           if (features.score >= 40) status = "danger";
           else if (features.score > 0) status = "warning";

           const listItems = features.details.map(item => `
               <li class="flex items-center space-x-3 ${item.risk === 'high' ? 'text-red-500 font-bold' : 'text-gray-600'}">
                   <i class="fas ${item.risk === 'high' ? 'fa-exclamation-triangle' : 'fa-info-circle'} w-5"></i>
                   <span>${item.msg}</span>
               </li>
           `);

           renderResult(
               status, 
               status === "danger" ? "악성 의심" : (status === "warning" ? "주의 요망" : "정상 링크"),
               "알고리즘 패턴 분석 결과입니다.",
               features.score, 
               listItems
           );
       }

       // --- 5. 화면 출력 제어 ---
       function renderResult(type, title, subtitle, score, listItems) {
           const card = document.getElementById('resultCard');
           const titleEl = document.getElementById('resultTitle');
           const scoreEl = document.getElementById('riskScore');
           const listEl = document.getElementById('featureList');
           
           // 색상 설정
           const colors = {
               danger: { border: 'border-red-500', text: 'text-red-600' },
               warning: { border: 'border-yellow-500', text: 'text-yellow-600' },
               safe: { border: 'border-green-500', text: 'text-green-600' }
           };

           card.className = `bg-white rounded-xl shadow-sm p-6 border-l-8 ${colors[type].border}`;
           titleEl.className = `text-2xl font-bold ${colors[type].text}`;
           scoreEl.className = `text-3xl font-bold ${colors[type].text}`;
           
           titleEl.innerText = title;
           document.getElementById('resultSubtitle').innerText = subtitle;
           scoreEl.innerText = score;
           listEl.innerHTML = listItems.length ? listItems.join('') : '<li class="text-green-600 font-medium"><i class="fas fa-shield-alt mr-2"></i>탐지된 위협 패턴이 없습니다.</li>';
           
           // 결과 창으로 스크롤
           window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });
       }
   </script>
</body>
</html>
