<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AYTAR Otonom Savunma Sistemi v2.0</title>
    <style>
        body {
            background-color: #050510;
            color: #00ff00;
            font-family: 'Courier New', Courier, monospace;
            margin: 0;
            overflow: hidden;
            display: flex;
            height: 100vh;
        }
        #radar-container {
            width: 65%;
            position: relative;
            border-right: 2px solid #004400;
            /* Arka plan gradyanı nebula renklerine uygun güncellendi */
            background: radial-gradient(circle at center, #0a0a2a 0%, #000000 100%);
        }
        #terminal-container {
            width: 35%;
            padding: 20px;
            box-sizing: border-box;
            background-color: #020205;
            display: flex;
            flex-direction: column;
        }
        h2 { margin-top: 0; color: #00ccff; border-bottom: 1px solid #00ccff; padding-bottom: 10px; }
        #log {
            flex-grow: 1;
            overflow-y: auto;
            font-size: 13px;
            line-height: 1.5;
            padding-right: 10px;
        }
        #log::-webkit-scrollbar { width: 8px; }
        #log::-webkit-scrollbar-thumb { background: #004400; border-radius: 4px; }
        .log-entry { margin-bottom: 5px; }
        .warning { color: #ff3333; font-weight: bold; }
        .action { color: #ffff00; }
        .success { color: #00ff00; }
        .info { color: #00ccff; }
        
        #radar-canvas {
            display: block;
            width: 100%;
            height: 100%;
        }
        .overlay-text {
            position: absolute;
            top: 20px;
            left: 20px;
            color: rgba(255,255,255,0.7);
            font-size: 13px;
            line-height: 1.6;
            pointer-events: none;
        }
        #stats { color: #00ff00; font-size: 16px; font-weight: bold; }
        #lock-status { color: #ff3333; font-weight: bold; }
        #orbit-info { color: #ffff00; font-size: 11px; }
    </style>
</head>
<body>

    <div id="radar-container">
        <div class="overlay-text">
            MİLLİ PLATFORM: TÜRKSAT 6A<br>
            AEROJEL TERMAL DURUM: <span style="color:#00ff00">STABİL (22.4°C)</span><br>
            EDGE AI: <span style="color:#00ccff">ÇOKLU ANGAJMAN (MAX 3 LOK)</span><br>
            AKTİF LOK (KİLİT): <span id="lock-status">0 / 3</span><br><br>
            BERTARAF EDİLEN TEHDİT: <span id="stats">0</span><br>
            <span id="orbit-info">YÖRÜNGE HIZI: 7.5 km/s | YÖN: KUZEY-DOĞU</span>
        </div>
        <canvas id="radar-canvas"></canvas>
    </div>

    <div id="terminal-container">
        <h2>AYTAR SİSTEM TERMİNALİ v2.0</h2>
        <div id="log"></div>
    </div>

    <script>
        const canvas = document.getElementById('radar-canvas');
        const ctx = canvas.getContext('2d');
        const logDiv = document.getElementById('log');
        const statsSpan = document.getElementById('stats');
        const lockSpan = document.getElementById('lock-status');

        function resize() {
            canvas.width = canvas.parentElement.clientWidth;
            canvas.height = canvas.parentElement.clientHeight;
            satellite.x = canvas.width / 2;
            satellite.y = canvas.height / 2;
            // Yıldızları yeniden ekran boyutuna göre dağıt
            spawnStars();
        }
        window.addEventListener('resize', resize);

        const satellite = { x: 0, y: 0 };
        
        // HAREKET VE ARKA PLAN DEĞİŞKENLERİ
        let starsList = []; // Arka plandaki kayan yıldızlar
        const SATELLITE_ORBIT_SPEED = 0.5; // Arka planın kayma hızı (Uydunun hızı)
        const orbitDirection = { x: 1, y: -0.5 }; // Uydu sağa-yukarı gidiyor (Kuzey-Doğu)
        
        let debrisList = []; 
        const MAX_DEBRIS_ON_SCREEN = 8; 
        const MAX_SIMULTANEOUS_LOCKS = 3; 
        let deflectedCount = 0;

        function addLog(message, type = '') {
            const time = new Date().toLocaleTimeString('tr-TR', { hour12: false, fractionalSecondDigits: 3 });
            const p = document.createElement('div');
            p.className = 'log-entry ' + type;
            p.innerHTML = `[${time}] > ${message}`;
            logDiv.appendChild(p);
            
            if (logDiv.childNodes.length > 35) {
                logDiv.removeChild(logDiv.firstChild);
            }
            logDiv.scrollTop = logDiv.scrollHeight; 
        }

        // --- ARKA PLAN VE HAREKET SİMÜLASYONU (Yeni) ---

        // Derinlik algısı için procedurally yıldızlar oluştur
        function spawnStars() {
            starsList = [];
            // Yaklaşık 200 yıldız yeterli
            for(let i=0; i<200; i++) {
                // z değeri derinliktir (hız ve boyutu belirler)
                let z = Math.random(); 
                starsList.push({
                    x: Math.random() * canvas.width,
                    y: Math.random() * canvas.height,
                    z: z,
                    // Derindeki (küçük z) yıldızlar yavaş, yakındaki (büyük z) hızlı kayar (Parallax)
                    size: 0.5 + z * 1.5,
                    speed: 0.1 + z * SATELLITE_ORBIT_SPEED 
                });
            }
        }

        function drawBackgroundStars() {
            // Yıldızların kayma yönü uydunun hareket yönünün TERSİDİR.
            const driftX = orbitDirection.x * -1;
            const driftY = orbitDirection.y * -1;

            starsList.forEach(star => {
                // Yıldızı hareket ettir
                star.x += driftX * star.speed;
                star.y += driftY * star.speed;

                // Ekrandan çıkarsa diğer taraftan sok (Infinite scrolling)
                if (star.x < 0) star.x = canvas.width;
                if (star.x > canvas.width) star.x = 0;
                if (star.y < 0) star.y = canvas.height;
                if (star.y > canvas.height) star.y = 0;

                // Çiz
                ctx.beginPath();
                ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2);
                // Derinliklerine göre opaklık ver (Derindekiler daha soluk)
                ctx.fillStyle = `rgba(255, 255, 255, ${0.1 + star.z * 0.7})`;
                ctx.fill();
            });
        }

        function drawNebulaBackground() {
            // Arka planda procedurally bulutsu şekilleri çiz
            ctx.shadowBlur = 0;

            // Nebula 1 (Mor)
            let grad1 = ctx.createRadialGradient(canvas.width*0.2, canvas.height*0.3, 0, canvas.width*0.2, canvas.height*0.3, 300);
            grad1.addColorStop(0, 'rgba(100, 0, 150, 0.15)');
            grad1.addColorStop(1, 'rgba(0,0,0,0)');
            ctx.fillStyle = grad1;
            ctx.fillRect(0,0, canvas.width, canvas.height);

            // Nebula 2 (Mavi)
            let grad2 = ctx.createRadialGradient(canvas.width*0.8, canvas.height*0.7, 0, canvas.width*0.8, canvas.height*0.7, 400);
            grad2.addColorStop(0, 'rgba(0, 50, 150, 0.15)');
            grad2.addColorStop(1, 'rgba(0,0,0,0)');
            ctx.fillStyle = grad2;
            ctx.fillRect(0,0, canvas.width, canvas.height);
        }

        // --- ENKAZ VE SAVUNMA SİMÜLASYONU (v1.0'dan korundu) ---

        function spawnDebris() {
            let edge = Math.floor(Math.random() * 4);
            let startX, startY;

            if (edge === 0) { startX = Math.random() * canvas.width; startY = -50; } 
            else if (edge === 1) { startX = canvas.width + 50; startY = Math.random() * canvas.height; } 
            else if (edge === 2) { startX = Math.random() * canvas.width; startY = canvas.height + 50; } 
            else { startX = -50; startY = Math.random() * canvas.height; } 

            let angleToSat = Math.atan2(satellite.y - startY, satellite.x - startX);
            angleToSat += (Math.random() - 0.5) * 0.5; // Daha fazla rastgelelik

            // Uydunun hareket hızı enkazların relative hızına eklenmeli
            // Basitlik için hızları biraz rastgele yapalım
            let speed = 1.0 + Math.random() * 2.0; 
            let id = Math.floor(Math.random() * 9000) + 1000;

            debrisList.push({
                id: id,
                x: startX,
                y: startY,
                speedX: Math.cos(angleToSat) * speed,
                speedY: Math.sin(angleToSat) * speed,
                size: 3 + Math.random() * 3,
                state: 'TRACKING',
                isLocked: false,
                distanceToSat: 9999,
                loggedWarning: false
            });
        }

        function drawRadar() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // --- ARKA PLAN KATMANLARI (Önce çizilir) ---
            drawNebulaBackground(); // Nebula bulutları
            drawBackgroundStars(); // Kayan yıldızlar (Hareketi gösterir)

            // --- HUD KATMANLARI ---
            
            // Uydu Gövdesi
            ctx.beginPath();
            ctx.arc(satellite.x, satellite.y, 12, 0, Math.PI * 2);
            ctx.fillStyle = '#00ccff';
            ctx.fill();
            ctx.shadowBlur = 25;
            ctx.shadowColor = '#00ccff';
            
            // Lazer Menzil Çemberi
            ctx.beginPath();
            ctx.arc(satellite.x, satellite.y, 220, 0, Math.PI * 2);
            ctx.strokeStyle = 'rgba(255, 0, 0, 0.3)';
            ctx.lineWidth = 1;
            ctx.stroke();
            ctx.shadowBlur = 0;

            // Çöplerin Çizimi
            debrisList.forEach(debris => {
                ctx.beginPath();
                ctx.arc(debris.x, debris.y, debris.size, 0, Math.PI * 2);
                
                if (debris.state === 'DEFLECTED') ctx.fillStyle = '#555555'; 
                else if (debris.isLocked) ctx.fillStyle = '#ff00ff'; // Kilitli hedef morumsu
                else ctx.fillStyle = '#ff3333'; 
                
                ctx.fill();

                // Hedef Vektörü (Kuyruk)
                ctx.beginPath();
                ctx.moveTo(debris.x, debris.y);
                ctx.lineTo(debris.x + (debris.speedX * 12), debris.y + (debris.speedY * 12));
                ctx.strokeStyle = debris.state === 'DEFLECTED' ? 'rgba(100, 100, 100, 0.3)' : 'rgba(255, 51, 51, 0.6)';
                ctx.stroke();

                // JÜRİ ŞOVU: Kilitlenilmiş hedeflerin etrafına Sniper Nişangahı çiz
                if (debris.isLocked) {
                    ctx.beginPath();
                    ctx.rect(debris.x - 10, debris.y - 10, 20, 20);
                    ctx.strokeStyle = '#ff0000';
                    ctx.lineWidth = 1;
                    ctx.stroke();
                    // Köşelere nişangah çizgileri
                    ctx.beginPath();
                    ctx.moveTo(debris.x, debris.y - 15); ctx.lineTo(debris.x, debris.y + 15);
                    ctx.moveTo(debris.x - 15, debris.y); ctx.lineTo(debris.x + 15, debris.y);
                    ctx.strokeStyle = 'rgba(255,0,0,0.5)';
                    ctx.stroke();

                    // Lazer Çizimi
                    ctx.beginPath();
                    ctx.moveTo(satellite.x, satellite.y);
                    ctx.lineTo(debris.x, debris.y);
                    ctx.strokeStyle = Math.random() > 0.2 ? '#00ff00' : '#ffffff'; 
                    ctx.lineWidth = 3;
                    ctx.stroke();
                    
                    // Plazma patlaması
                    ctx.beginPath();
                    ctx.arc(debris.x, debris.y, debris.size + Math.random() * 10, 0, Math.PI * 2);
                    ctx.fillStyle = 'rgba(255, 255, 0, 0.8)';
                    ctx.fill();
                }
            });
        }

        function updateLogic() {
            // Enkaz sayısını koru
            while (debrisList.length < MAX_DEBRIS_ON_SCREEN) {
                spawnDebris();
            }

            // O anki kilitli hedef sayısını say
            let currentLocks = debrisList.filter(d => d.isLocked).length;
            lockSpan.innerText = `${currentLocks} / ${MAX_SIMULTANEOUS_LOCKS}`;

            for (let i = debrisList.length - 1; i >= 0; i--) {
                let d = debrisList[i];
                
                let prevDist = Math.hypot(d.x - satellite.x, d.y - satellite.y);
                d.x += d.speedX;
                d.y += d.speedY;
                d.distanceToSat = Math.hypot(d.x - satellite.x, d.y - satellite.y);
                
                let isApproaching = d.distanceToSat < prevDist;

                // Ekrandan çıkanları sil
                if (d.x < -300 || d.x > canvas.width + 300 || d.y < -300 || d.y > canvas.height + 300) {
                    debrisList.splice(i, 1);
                    continue;
                }

                // 220px Menziline girdi mi?
                if (d.distanceToSat <= 220 && d.state === 'TRACKING' && isApproaching) {
                    // EĞER kilitli değilse ve KİLİT KAPASİTEMİZ dolmadıysa kilitlen!
                    if (!d.isLocked && currentLocks < MAX_SIMULTANEOUS_LOCKS) {
                        d.isLocked = true;
                        currentLocks++;
                        addLog(`HEDEF ANGAJE: Obje #${d.id} Kalman filtresi kilitlendi.`, "warning");
                    }
                }

                // Kilitliyse Lazerin ablasyon itkisi uygula
                if (d.isLocked) {
                    let pushAngle = Math.atan2(d.y - satellite.y, d.x - satellite.x);
                    d.speedX += Math.cos(pushAngle) * 0.12; // İtme gücü
                    d.speedY += Math.sin(pushAngle) * 0.12;

                    if (Math.random() < 0.04) { 
                        addLog(`Lazer Aktif: Obje #${d.id} saptırılıyor...`, "action");
                    }
                }

                // Saptırıldı mı kontrolü (Artık bizden uzaklaşıyorsa)
                if (!isApproaching && d.state === 'TRACKING' && d.distanceToSat <= 220) {
                    d.state = 'DEFLECTED';
                    if (d.isLocked) {
                        d.isLocked = false; // Kilidi bırak!
                        currentLocks--;
                    }
                    deflectedCount++;
                    statsSpan.innerText = deflectedCount;
                    addLog(`BERTARAF: Obje #${d.id} yörüngeden saptırıldı.`, "success");
                }
            }
        }

        function loop() {
            updateLogic();
            drawRadar();
            requestAnimationFrame(loop);
        }

        // --- REFERANSLAR (APA 7 - Raporda kullanılmak üzere) ---
        // CelesTrak. (t.y.). Current NORAD two-line element sets. https://celestrak.org/NORAD/elements/
        // Redmon, J. vd. (2016). You only look once: Unified, real-time object detection. Proc. CVPR.

        // Başlangıç
        resize(); // resize içinde spawnStars çağrılır
        addLog("AYTAR v2.0 İşletim Sistemi Başlatıldı.");
        addLog(`Yörünge Taraması Aktif. Max Angajman Kapasitesi: ${MAX_SIMULTANEOUS_LOCKS}.`, "info");
        loop();

    </script>
</body>
</html>
