<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>最终定版：多线程 + 10GB 大文件持续测速</title>
    
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    <style>
        /* CSS 部分：高级轻奢极简风格 */
        body {
            font-family: 'Inter', -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            margin: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background-color: #f2f4f8; 
            color: #333; 
            position: relative;
            overflow: hidden;
        }

        .container {
            background-color: #ffffff; 
            border-radius: 16px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.08); 
            padding: 40px 50px;
            width: 380px;
            text-align: center;
            transition: all 0.3s ease;
            position: relative;
            z-index: 10;
        }

        h1 {
            font-size: 1.8em;
            font-weight: 500; 
            color: #2c3e50; 
            margin-bottom: 35px;
            letter-spacing: 0.5px;
        }

        .data-row {
            margin-bottom: 25px; 
            padding: 12px 0;
            border-bottom: 1px solid #e0e6ed; 
        }
        .data-row:last-of-type {
            border-bottom: none;
        }

        .label {
            font-size: 0.9em;
            opacity: 0.75;
            color: #555;
            display: block;
            margin-bottom: 8px;
            font-weight: 400;
        }

        .speed-value {
            font-size: 3.2em; 
            font-weight: 700; 
            line-height: 1.1;
            color: #007bff; 
            display: inline-block; 
            vertical-align: baseline;
        }

        .unit {
            font-size: 0.6em; 
            font-weight: 500;
            vertical-align: super; 
            margin-left: 5px;
            opacity: 0.8;
            color: #666;
        }

        /* 历史最高峰值使用不同颜色和字体大小强调 */
        #absoluteMaxSpeedValue {
            color: #ff9900; /* 橙色强调 */
            font-size: 2.5em; 
        }

        .control-btn {
            padding: 14px 35px;
            font-size: 1.05em;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
            transition: all 0.3s ease;
            margin: 8px;
            min-width: 120px;
        }

        #btn_speed {
            background-color: #007bff; 
            color: #ffffff;
            box-shadow: 0 4px 15px rgba(0, 123, 255, 0.3);
        }

        #btn_speed:hover:not(:disabled) {
            background-color: #0056b3; 
            box-shadow: 0 6px 20px rgba(0, 123, 255, 0.4);
        }
        
        #stopBtn {
            background-color: #e9ecef; 
            color: #495057;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
        }
        
        #stopBtn:hover:not(:disabled) {
            background-color: #dae0e5; 
            color: #212529;
        }
        
        .control-btn:disabled {
            opacity: 0.6;
            cursor: not-allowed;
            box-shadow: none;
        }

        #statusMessage {
            margin-top: 30px;
            font-size: 0.95em;
            min-height: 20px;
            color: #6c757d; 
            line-height: 1.4;
        }

        .background-dots {
            position: absolute;
            width: 100vw;
            height: 100vh;
            top: 0;
            left: 0;
            background: radial-gradient(circle, rgba(0,0,0,0.03) 1px, transparent 1px);
            background-size: 30px 30px;
            opacity: 0.6;
            z-index: 1;
            pointer-events: none;
        }

        /* 隐藏原始页面的微小数据显示区域，只显示在我们的卡片上 */
        #result_down, #result_up, #result_downs, #result_ups, #delay_time, #shake_num, #lost_num {
            display: none; 
        }

    </style>
</head>
<body>
    <div class="background-dots"></div> 
    <div class="container">
        <h1>持续峰值测速 (10GB)</h1>

        <div class="data-row">
            <span class="label">实时下载速度（并发总和）</span>
            <span id="currentSpeed" class="speed-value">0.00</span>
            <span class="unit">Mbps</span>
        </div>

        <div class="data-row">
            <span class="label">本次测试峰值</span>
            <span id="roundMaxSpeedValue" class="speed-value" style="font-size: 2.2em; color: #6c757d;">0.00</span>
            <span class="unit">Mbps</span>
        </div>
        
        <div class="data-row">
            <span class="label">历史最高峰值</span>
            <span id="absoluteMaxSpeedValue" class="speed-value">0.00</span>
            <span class="unit">Mbps</span>
        </div>

        <div>
            <input type="button" id="btn_speed" class="control-btn" value="开始持续测速" onclick="startnet(3)">
            <button id="stopBtn" class="control-btn" onclick="stopnet()" disabled>停止</button>
        </div>

        <p id="delay_time"></p><p id="shake_num"></p><p id="lost_num"></p>
        <p id="result_down" class="sults"></p><p id="result_up" class="sults"></p>
        <p id="result_downs"></p><p id="result_ups"></p>

        <p id="statusMessage">点击“开始持续测速”以启动 10GB 大文件测试。</p>
    </div>

    <script>
        // *** 核心配置：已更新为 10GB Cloudflare 测试链接 ***
        const BASE_URL = "https://speed.cloudflare.com/__down?bytes=10737418240"; // 10 GB
        const NUM_CONNECTIONS = 1000; 
        const TOTAL_BYTES = 10737418240; // 10 GB
        
        // 新增/修改的全局状态变量
        let roundMaxSpeed = 0;      // 当前轮次的最高峰值
        let absoluteMaxSpeed = 0;   // 历史最高峰值
        let currentRound = 0;       // 当前轮次计数 (在这个版本中，它将保持为 1，因为不再循环)
        let isTesting = false;      // 测试进行中（用于控制 requestAnimationFrame 循环）
        let startTime = 0;
        let xhrArray = []; 
        let connectionLoadedBytes = new Array(NUM_CONNECTIONS).fill(0);

        function updateStatus(message, color = '#6c757d') {
            const statusElement = document.getElementById('statusMessage');
            statusElement.textContent = message;
            statusElement.style.color = color;
        }
        
        function updateSpeedDisplay() {
            if (!isTesting) return;

            const currentTime = new Date().getTime();
            const totalTime = currentTime - startTime; // 总耗时 (毫秒)
            
            // 计算所有连接的总下载字节数
            const totalLoaded = connectionLoadedBytes.reduce((sum, bytes) => sum + bytes, 0); 
            
            // 只有在时间大于 0 时才计算速度，防止 NaN
            if (totalTime > 0) {
                // 计算实时速度 (Mbps)
                // (总字节 / 总耗时 ms) * 8 (位/字节) * 1000 (ms/s) / 1048576 (字节/Mb)
                const currentSpeedMbps = (totalLoaded / totalTime) * 8 * 1000 / 1048576; 
                
                // 更新当前速度显示
                document.getElementById('currentSpeed').textContent = currentSpeedMbps.toFixed(2);
                
                // 更新本次测试的最高速度
                if (currentSpeedMbps > roundMaxSpeed) {
                    roundMaxSpeed = currentSpeedMbps;
                    document.getElementById('roundMaxSpeedValue').textContent = roundMaxSpeed.toFixed(2);
                    
                    // 同时更新历史最高峰值
                    if (roundMaxSpeed > absoluteMaxSpeed) {
                        absoluteMaxSpeed = roundMaxSpeed;
                        document.getElementById('absoluteMaxSpeedValue').textContent = absoluteMaxSpeed.toFixed(2);
                    }
                }
            }


            // 更新状态信息
            const targetTotalBytes = TOTAL_BYTES * NUM_CONNECTIONS;
            const progress = totalLoaded / targetTotalBytes; 
            const progressPercent = (progress * 100).toFixed(6); // 进度使用更多小数位

            updateStatus(`测速进行中 (进度: ${progressPercent}%, 已下载: ${(totalLoaded / 1048576).toFixed(2)} MB)`, 'rgb(0, 123, 255)');

            // 持续更新速度，直到被 stopnet 停止或所有文件下载完成
            requestAnimationFrame(updateSpeedDisplay);
        }

        // 核心测速启动函数
        function startnet(nodeId) {
            // 如果已经在测试中，则忽略
            if (isTesting) {
                return;
            }
            
            currentRound = 1; // 这是一个单次运行，设置为第 1 轮

            // 按钮和状态设置
            document.getElementById('btn_speed').disabled = true;
            document.getElementById('btn_speed').value = `测速进行中...`;
            document.getElementById('stopBtn').disabled = false;
            
            isTesting = true; // 开启测试状态

            // 重置本轮状态
            roundMaxSpeed = 0;
            // absoluteMaxSpeed 不重置，保留历史最高值
            connectionLoadedBytes.fill(0);
            document.getElementById('currentSpeed').textContent = '0.00';
            document.getElementById('roundMaxSpeedValue').textContent = '0.00'; 
            updateStatus(`正在建立 ${NUM_CONNECTIONS} 个连接...`, 'rgb(255, 165, 0)'); 

            // 清空旧的连接并准备新连接
            xhrArray = []; 
            startTime = new Date().getTime();
            
            let completedConnections = 0;
            const totalConnections = NUM_CONNECTIONS;

            for (let i = 0; i < totalConnections; i++) {
                const xhr = new XMLHttpRequest();
                xhrArray.push(xhr);

                // 每个连接使用一个随机参数
                const uniqueUrl = BASE_URL + `&conn=${i}&r=${Date.now()}`;
                
                xhr.open('GET', uniqueUrl, true);
                
                // 监听进度事件
                xhr.onprogress = function(event) {
                    if (!isTesting) return; 
                    connectionLoadedBytes[i] = event.loaded;
                };

                // 监听加载完成事件
                xhr.onload = function() {
                    if (xhr.status === 200) {
                        completedConnections++;
                        // 当所有连接都完成后，测试结束
                        if (completedConnections === totalConnections) {
                            finishTest(true);
                        }
                    } else {
                        // 如果有任何连接失败，则终止测试
                        updateStatus(`测速失败，连接 ${i} 出现 HTTP 状态码: ${xhr.status}。测试已终止。`, 'red');
                        stopnet(false); 
                    }
                };

                // 监听错误/超时事件
                xhr.onerror = xhr.ontimeout = function() {
                    updateStatus(`网络请求出错/超时！测试已终止。`, 'red'); 
                    stopnet(false);
                };
                
                xhr.send();
            }

            // 启动速度显示循环
            requestAnimationFrame(updateSpeedDisplay);
        }

        // 自动完成/停止测试
        function finishTest(isAuto = false) {
            // 设置 isTesting = false 确保 updateSpeedDisplay 循环停止
            isTesting = false;
            
            // 停止所有并发连接
            xhrArray.forEach(xhr => xhr.abort());

            if (isAuto) {
                // 如果是自动完成 (10GB下载完)，显示最终结果
                updateStatus(`10GB 文件下载完成！最终最高速度: ${absoluteMaxSpeed.toFixed(2)} Mbps`, 'green');
            } else {
                // 如果是手动停止或出错
                updateStatus(`已停止测速。本次最高速度: ${absoluteMaxSpeed.toFixed(2)} Mbps`, 'red');
            }
            
            // 重置控制按钮状态
            resetControls();
        }

        // 手动停止测试
        function stopnet(isManual = true) {
            // 如果测试已停止，且不是手动点击停止（例如：自动完成后的调用），则忽略
            if (!isTesting && !isManual) return; 

            finishTest(false);
        }
        
        function resetControls() {
            document.getElementById('btn_speed').disabled = false;
            document.getElementById('btn_speed').value = '再次测速';
            document.getElementById('stopBtn').disabled = true;
            currentRound = 0; // 重置轮次计数
        }

        // 确保页面加载完成后，按钮状态是正确的
        document.addEventListener('DOMContentLoaded', () => {
            document.getElementById('stopBtn').disabled = true;
        });

    </script>
</body>
</html>                        return response.blob(); 
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
