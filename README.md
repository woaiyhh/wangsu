<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>极简多线程测速 (优化版)</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 100vh; margin: 0; background-color: #f4f4f9; }
        .container { background: white; padding: 40px; border-radius: 12px; box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1); text-align: center; width: 450px; }
        h1 { color: #333; margin-bottom: 30px; }
        button { padding: 12px 25px; font-size: 1.1em; background-color: #007bff; color: white; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.3s; }
        button:hover { background-color: #0056b3; }
        .result-box { margin-top: 25px; text-align: left; }
        .result-item { margin-bottom: 15px; padding: 10px; border: 1px solid #e0e0e0; border-radius: 4px; background-color: #fafafa; }
        .result-item strong { display: inline-block; width: 150px; color: #555; }
        .speed-value { font-size: 1.4em; font-weight: bold; color: #28a745; margin-left: 10px; }
        .log-message { margin-top: 15px; color: #dc3545; font-size: 0.9em; min-height: 20px;}
        .warning { color: orange; font-weight: bold; }
    </style>
</head>
<body>

    <div class="container">
        <h1>多线程极速测速</h1>
        <p class="log-message warning" id="log">请先点击“开始测速”按钮。</p>
        
        <button id="startBtn" onclick="startTest()">开始测速</button>

        <div class="result-box">
            <div id="server-info" class="result-item"><strong>测试服务器:</strong> <span class="speed-value" id="server-value">国内默认节点</span></div>
            <div id="ping-result" class="result-item"><strong>网络延迟 (Ping):</strong> <span class="speed-value">N/A</span></div>
            <div id="download-result" class="result-item"><strong>下载速度:</strong> <span class="speed-value">N/A</span></div>
            <div id="upload-result" class="result-item"><strong>上传速度:</strong> <span class="speed-value">N/A</span></div>
        </div>
        
    </div>

    <script>
        // *******************************************************************
        // 核心配置
        // *******************************************************************
        // 硬编码一个国内常用的、高性能的 Ookla 兼容测试服务器地址。
        // 这是为了解决 API 不稳定和跨域问题。该服务器地址用于测试数据流。
        const TEST_SERVER_BASE_URL = 'http://speedtest.gz.chinamobile.com/speedtest/'; 
        
        // 并发连接数 (多线程)。这是测出最高速度的关键。
        const CONCURRENT_THREADS = 4; 
        
        // 每个线程下载和上传的数据量 (MB)。测试数据量越大，结果越稳定。
        const TEST_SIZE_MB_PER_THREAD = 8; 
        const TEST_SIZE_BYTES = TEST_SIZE_MB_PER_THREAD * 1024 * 1024;
        
        // *******************************************************************

        const btn = document.getElementById('startBtn');
        const logElement = document.getElementById('log');

        function updateUI(type, value) {
            document.getElementById(`${type}-result`).querySelector('.speed-value').textContent = value;
        }

        function updateLog(message, isError = false) {
            logElement.textContent = `当前状态: ${message}`;
            logElement.classList.toggle('warning', isError);
        }

        /**
         * 检查当前环境是否安全可靠（即非本地文件）
         */
        function checkEnvironment() {
            if (window.location.protocol === 'file:') {
                updateLog('警告：当前是本地文件运行！可能会因 CORS 限制而失败。请部署到 Web 服务器上运行！', true);
                // 允许继续尝试，但用户已被警告
            }
        }

        /**
         * 延迟/Ping 测试
         */
        async function runPingTest() {
            updateLog("正在进行网络延迟 (Ping) 测试...");
            updateUI('ping', '测试中...');
            const pings = [];
            const PING_COUNT = 3; 
            
            // 使用 Ookla 兼容的 latency.txt 文件进行 Ping 测试
            const PING_URL = `${TEST_SERVER_BASE_URL}latency.txt?x=${Math.random()}`; 

            for (let i = 0; i < PING_COUNT; i++) {
                const startTime = performance.now();
                try {
                    // 使用 no-store 确保每次都是新请求
                    await fetch(PING_URL, { method: 'GET', cache: 'no-store' });
                    const endTime = performance.now();
                    pings.push(endTime - startTime);
                } catch (e) {
                    console.error("Ping Test Failed:", e);
                    // 记录错误但继续
                }
            }

            if (pings.length === 0) {
                 updateUI('ping', '错误/超时');
                 return -1;
            }

            const averagePing = pings.reduce((a, b) => a + b) / pings.length;
            const resultValue = averagePing.toFixed(2) + ' ms';
            updateUI('ping', resultValue);
            return averagePing;
        }

        /**
         * 下载测速：使用多个并发连接下载数据
         */
        async function runDownloadTest() {
            updateLog(`正在进行 ${CONCURRENT_THREADS} 线程下载测速...`);
            updateUI('download', '测试中...');
            const downloadPromises = [];
            const startTime = performance.now();
            
            // Ookla 兼容服务器通常提供 random[size]mb.bin 文件
            const DOWNLOAD_URL_TEMPLATE = `${TEST_SERVER_BASE_URL}random${TEST_SIZE_MB_PER_THREAD}mb.bin`;

            for (let i = 0; i < CONCURRENT_THREADS; i++) {
                const url = `${DOWNLOAD_URL_TEMPLATE}?x=${Math.random()}`;
                
                // 核心：并发发起 fetch 请求
                const promise = fetch(url, { method: 'GET', cache: 'no-store', mode: 'cors' })
                    .then(response => {
                        if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
                        return response.blob(); 
                    });
                downloadPromises.push(promise);
            }

            try {
                // 等待所有并发请求完成
                await Promise.all(downloadPromises);
                
                const endTime = performance.now();
                const durationSeconds = (endTime - startTime) / 1000;
                
                const totalBytes = CONCURRENT_THREADS * TEST_SIZE_BYTES; 
                
                // 转换为 Mbps: (Bytes * 8) / (1024 * 1024) / Seconds
                const speedMbps = (totalBytes * 8) / (durationSeconds * 1024 * 1024);
                
                const resultValue = speedMbps.toFixed(2) + ' Mbps';
                updateUI('download', resultValue);
                return speedMbps;

            } catch (e) {
                console.error("下载测试失败:", e);
                updateLog(`下载失败，请检查服务器连接或 CORS 错误: ${e.message}`, true);
                updateUI('download', '失败');
                return 0;
            }
        }

        /**
         * 上传测速：使用多个并发连接上传随机数据
         */
        async function runUploadTest() {
            updateLog(`正在进行 ${CONCURRENT_THREADS} 线程上传测速...`);
            updateUI('upload', '测试中...');
            
            // 创建随机数据 Blob
            const data = new Uint8Array(TEST_SIZE_BYTES);
            for (let i = 0; i < TEST_SIZE_BYTES; i++) {
                data[i] = Math.floor(Math.random() * 256);
            }
            const uploadData = new Blob([data], { type: 'application/octet-stream' }); 
            
            const uploadPromises = [];
            const startTime = performance.now();
            
            // Ookla 兼容服务器通常使用 upload.php 或类似的脚本
            const UPLOAD_URL = `${TEST_SERVER_BASE_URL}upload.php`;

            for (let i = 0; i < CONCURRENT_THREADS; i++) {
                const url = `${UPLOAD_URL}?x=${Math.random()}`;
                
                // 核心：并发发起 POST 请求上传数据
                const promise = fetch(url, {
                    method: 'POST',
                    body: uploadData, 
                    cache: 'no-store',
                    mode: 'cors'
                })
                .then(response => {
                    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
                    return response.text();
                });
                uploadPromises.push(promise);
            }

            try {
                await Promise.all(uploadPromises);
                
                const endTime = performance.now();
                const durationSeconds = (endTime - startTime) / 1000;
                const totalBytes = CONCURRENT_THREADS * TEST_SIZE_BYTES; 
                
                // 转换为 Mbps
                const speedMbps = (totalBytes * 8) / (durationSeconds * 1024 * 1024);
                
                const resultValue = speedMbps.toFixed(2) + ' Mbps';
                updateUI('upload', resultValue);
                return speedMbps;

            } catch (e) {
                console.error("上传测试失败:", e);
                updateLog(`上传失败，请检查服务器连接或 CORS 错误: ${e.message}`, true);
                updateUI('upload', '失败');
                return 0;
            }
        }

        /**
         * 启动测速流程
         */
        async function startTest() {
            btn.disabled = true;
            btn.textContent = '测速进行中...';
            updateUI('ping', 'N/A');
            updateUI('download', 'N/A');
            updateUI('upload', 'N/A');
            document.getElementById('server-value').textContent = '中国移动 (广州)';
            
            // 确保环境检查和警告
            checkEnvironment();

            try {
                // 1. Ping 测试
                await runPingTest();

                // 2. 下载测速
                await runDownloadTest();

                // 3. 上传测速
                await runUploadTest();
                
                updateLog("所有测试已完成！结果可能受限于服务器或网络路径。", false);
                
            } catch (error) {
                // 意外错误捕获
                updateLog("测速流程因未知错误中断，请检查控制台。", true);
            }
            
            btn.textContent = '测速完成，重新开始';
            btn.disabled = false;
        }

    </script>

</body>
</html>
