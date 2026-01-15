<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Particle House - Hand Gesture Control</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #050510; font-family: sans-serif; }
        canvas { display: block; }
        #loading {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: #ff69b4; font-size: 24px; text-align: center;
            text-shadow: 0 0 10px #ff69b4; pointer-events: none; z-index: 10;
        }
        #video-container {
            position: absolute; bottom: 20px; right: 20px;
            width: 160px; height: 120px; border-radius: 10px; overflow: hidden;
            border: 2px solid #ff69b4; opacity: 0.7; z-index: 5;
            transition: border-color 0.3s ease;
        }
        video { width: 100%; height: 100%; object-fit: cover; transform: scaleX(-1); }
        .instructions {
            position: absolute; top: 20px; left: 20px; color: rgba(255,255,255,0.8);
            pointer-events: none; line-height: 1.6;
        }
        .key { color: #ff69b4; font-weight: bold; }
        /* 底部文字样式 */
        .footer-text {
            position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%);
            color: #ff69b4; font-size: 20px; text-align: center;
            text-shadow: 0 0 15px #ff69b4; pointer-events: none; z-index: 10;
            font-weight: bold;
        }
    </style>
    
    <!-- Three.js & Post Processing -->
    <script src="https://unpkg.com/three@0.132.2/build/three.min.js"></script>
    <script src="https://unpkg.com/three@0.132.2/examples/js/postprocessing/EffectComposer.js"></script>
    <script src="https://unpkg.com/three@0.132.2/examples/js/postprocessing/RenderPass.js"></script>
    <script src="https://unpkg.com/three@0.132.2/examples/js/postprocessing/ShaderPass.js"></script>
    <script src="https://unpkg.com/three@0.132.2/examples/js/shaders/CopyShader.js"></script>
    <script src="https://unpkg.com/three@0.132.2/examples/js/shaders/LuminosityHighPassShader.js"></script>
    <script src="https://unpkg.com/three@0.132.2/examples/js/postprocessing/UnrealBloomPass.js"></script>
    <script src="https://unpkg.com/three@0.132.2/examples/js/controls/OrbitControls.js"></script>

    <!-- MediaPipe Hands -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
</head>
<body>
    <div id="loading">正在初始化视觉引擎与摄像头...<br><span style="font-size:14px">请允许摄像头权限</span></div>
    
    <div class="instructions">
        <div>✋ <span class="key">张开手掌</span> : 粒子爆炸 (Explode)</div>
        <div>✊ <span class="key">握紧拳头/无手势</span> : 红顶粉房 (House)</div>
        <div>✌️ <span class="key">比耶手势</span> : 跳舞小人 (Dancing)</div>
    </div>

    <!-- 底部永久显示的文字 -->
    <div class="footer-text">@about编辑部 《好久没聚会了》</div>

    <div id="video-container">
        <video id="input_video" autoplay muted playsinline></video>
    </div>

<script>
    // --- 配置参数 ---
    const PARTICLE_COUNT = 25000;
    const HOUSE_BODY_COLOR = 0xffa6c9; // 粉色房体
    const HOUSE_ROOF_COLOR = 0xff0000; // 正红色屋顶
    const WINDOW_COLOR = 0xff0000;     // 红色窗户
    const EXPLOSION_RADIUS = 80;
    const TRANSITION_SPEED = 0.12; // 提高过渡速度，让回归更流畅
    
    // 状态定义
    const STATE_HOUSE = 0;
    const STATE_EXPLODE = 1;
    const STATE_DANCING = 2; // 新增：跳舞状态
    let currentState = STATE_HOUSE; // 默认房子状态
    let previousState = STATE_HOUSE; // 记录上一状态，用于平滑过渡

    // 粒子类型标记（用于区分房体/屋顶/窗户）
    let particleTypes = []; // 0=房体, 1=屋顶, 2=窗户

    // --- 跳舞小人相关变量 ---
    let dancer1, dancer2; // 两个跳舞的小人
    let danceGroup; // 小人组，用于整体旋转
    let showDancers = false; // 是否显示跳舞小人

    // --- Three.js 初始化 ---
    const scene = new THREE.Scene();
    scene.fog = new THREE.FogExp2(0x050510, 0.02);

    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 5, 30);

    const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.toneMapping = THREE.ReinhardToneMapping;
    document.body.appendChild(renderer.domElement);

    // --- 轨道控制器 ---
    const controls = new THREE.OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.autoRotate = true;
    controls.autoRotateSpeed = 0.5;
    controls.enablePan = false;
    controls.enableZoom = true;

    // --- 生成粒子纹理 ---
    function createSprite() {
        const canvas = document.createElement('canvas');
        canvas.width = 32; canvas.height = 32;
        const context = canvas.getContext('2d');
        const gradient = context.createRadialGradient(16, 16, 0, 16, 16, 16);
        gradient.addColorStop(0, 'rgba(255, 255, 255, 1)');
        gradient.addColorStop(0.2, 'rgba(255, 200, 220, 0.8)');
        gradient.addColorStop(0.5, 'rgba(255, 100, 150, 0.2)');
        gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
        context.fillStyle = gradient;
        context.fillRect(0, 0, 32, 32);
        const texture = new THREE.Texture(canvas);
        texture.needsUpdate = true;
        return texture;
    }

    // --- 创建姜饼人样式的跳舞小人 ---
    function createGingerbreadMan(color) {
        const manGroup = new THREE.Group();
        
        // 身体（椭圆）
        const bodyGeometry = new THREE.CylinderGeometry(1.2, 1.5, 4, 16);
        const bodyMaterial = new THREE.MeshBasicMaterial({ 
            color: color,
            wireframe: false,
            transparent: true,
            opacity: 0.9
        });
        const body = new THREE.Mesh(bodyGeometry, bodyMaterial);
        body.position.y = 2;
        manGroup.add(body);
        
        // 头部（圆形）
        const headGeometry = new THREE.SphereGeometry(1.2, 16, 16);
        const headMaterial = new THREE.MeshBasicMaterial({ 
            color: color,
            transparent: true,
            opacity: 0.9
        });
        const head = new THREE.Mesh(headGeometry, headMaterial);
        head.position.y = 5.5;
        manGroup.add(head);
        
        // 眼睛
        const eyeGeometry = new THREE.SphereGeometry(0.15, 8, 8);
        const eyeMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
        
        const leftEye = new THREE.Mesh(eyeGeometry, eyeMaterial);
        leftEye.position.set(-0.4, 6, 0.8);
        manGroup.add(leftEye);
        
        const rightEye = new THREE.Mesh(eyeGeometry, eyeMaterial);
        rightEye.position.set(0.4, 6, 0.8);
        manGroup.add(rightEye);
        
        // 嘴巴（微笑）
        const mouthPoints = [];
        for (let i = 0; i <= 10; i++) {
            const angle = Math.PI * 0.2 * (i / 10);
            mouthPoints.push(new THREE.Vector2(
                -0.5 + i * 0.1,
                -0.2 + Math.sin(angle) * 0.2
            ));
        }
        const mouthGeometry = new THREE.BufferGeometry().setFromPoints(mouthPoints);
        const mouthMaterial = new THREE.LineBasicMaterial({ color: 0x000000 });
        const mouth = new THREE.Line(mouthGeometry, mouthMaterial);
        mouth.position.set(0, 5.2, 0.9);
        manGroup.add(mouth);
        
        // 手臂
        const armGeometry = new THREE.CylinderGeometry(0.3, 0.3, 3, 8);
        const armMaterial = new THREE.MeshBasicMaterial({ 
            color: color,
            transparent: true,
            opacity: 0.9
        });
        
        const leftArm = new THREE.Mesh(armGeometry, armMaterial);
        leftArm.position.set(-1.8, 3.5, 0);
        leftArm.rotation.z = Math.PI / 4;
        manGroup.add(leftArm);
        
        const rightArm = new THREE.Mesh(armGeometry, armMaterial);
        rightArm.position.set(1.8, 3.5, 0);
        rightArm.rotation.z = -Math.PI / 4;
        manGroup.add(rightArm);
        
        // 腿
        const legGeometry = new THREE.CylinderGeometry(0.4, 0.4, 2.5, 8);
        const legMaterial = new THREE.MeshBasicMaterial({ 
            color: color,
            transparent: true,
            opacity: 0.9
        });
        
        const leftLeg = new THREE.Mesh(legGeometry, legMaterial);
        leftLeg.position.set(-0.6, -0.5, 0);
        manGroup.add(leftLeg);
        
        const rightLeg = new THREE.Mesh(legGeometry, legMaterial);
        rightLeg.position.set(0.6, -0.5, 0);
        manGroup.add(rightLeg);
        
        // 添加装饰
        const decorationGeometry = new THREE.SphereGeometry(0.1, 8, 8);
        const decorationMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
        
        // 身体装饰
        for (let i = 0; i < 5; i++) {
            const deco = new THREE.Mesh(decorationGeometry, decorationMaterial);
            deco.position.set(
                (Math.random() - 0.5) * 1,
                2 + i * 0.8,
                (Math.random() - 0.5) * 1
            );
            manGroup.add(deco);
        }
        
        return manGroup;
    }

    // --- 初始化跳舞小人 ---
    function initDancers() {
        // 创建跳舞组
        danceGroup = new THREE.Group();
        scene.add(danceGroup);
        
        // 创建两个不同颜色的姜饼人
        dancer1 = createGingerbreadMan(0xe67e22); // 橙色姜饼人
        dancer2 = createGingerbreadMan(0x9b59b6); // 紫色姜饼人
        
        // 设置初始位置
        dancer1.position.set(-3, 0, 0);
        dancer2.position.set(3, 0, 0);
        
        danceGroup.add(dancer1);
        danceGroup.add(dancer2);
        
        // 默认隐藏
        danceGroup.visible = false;
    }

    // --- 粒子系统数据 ---
    const posCurrent = new Float32Array(PARTICLE_COUNT * 3);
    const posHouse = new Float32Array(PARTICLE_COUNT * 3); // 房子目标位置
    const posExplode = new Float32Array(PARTICLE_COUNT * 3); // 爆炸目标位置
    const colors = new Float32Array(PARTICLE_COUNT * 3); // 粒子颜色数组
    // 保存初始颜色，用于爆炸后恢复
    const originalColors = new Float32Array(PARTICLE_COUNT * 3);

    // 1. 生成房子形态数据 (算法生成)
    function generateHouse() {
        const width = 14, height = 8, depth = 10;
        const roofHeight = 6;
        particleTypes = []; // 重置粒子类型
        
        for (let i = 0; i < PARTICLE_COUNT; i++) {
            let x, y, z;
            let particleType = 0; // 默认房体
            let valid = false;

            while (!valid) {
                // 随机选择生成部分：主体墙壁(65%) 或 屋顶(30%) 或 窗户(5%)
                const rand = Math.random();
                if (rand < 0.65) {
                    // 主体立方体（粉色房体）
                    x = (Math.random() - 0.5) * width;
                    y = (Math.random() - 0.5) * height;
                    z = (Math.random() - 0.5) * depth;
                    
                    // 避开窗户区域（让窗户位置空出来）
                    if (z > depth/2 - 1 && x > -3 && x < 3 && y > -2 && y < 2) {
                        continue; // 跳过窗户区域，留给窗户粒子
                    }
                    particleType = 0;
                } else if (rand < 0.95) {
                    // 屋顶（正红色）
                    const h = Math.random(); // 0-1
                    y = (height / 2) + h * roofHeight;
                    const scale = 1.0 - h; // 顶部收缩
                    x = (Math.random() - 0.5) * width * scale; 
                    z = (Math.random() - 0.5) * depth;
                    particleType = 1;
                } else {
                    // 窗户（红色，在房子正面）
                    x = (Math.random() - 0.5) * 6; // 窗户宽度-3~3
                    y = (Math.random() - 0.5) * 4; // 窗户高度-2~2
                    z = depth/2 - 0.5; // 窗户在房子正面
                    particleType = 2;
                }
                valid = true;
            }

            // 保存位置数据
            posHouse[i * 3] = x;
            posHouse[i * 3 + 1] = y;
            posHouse[i * 3 + 2] = z;

            // 初始位置设为房子位置
            posCurrent[i * 3] = x;
            posCurrent[i * 3 + 1] = y;
            posCurrent[i * 3 + 2] = z;

            // 保存粒子类型
            particleTypes.push(particleType);

            // 设置粒子颜色
            let color;
            if (particleType === 0) {
                // 粉色房体
                color = new THREE.Color(HOUSE_BODY_COLOR);
            } else if (particleType === 1) {
                // 正红色屋顶
                color = new THREE.Color(HOUSE_ROOF_COLOR);
            } else {
                // 红色窗户
                color = new THREE.Color(WINDOW_COLOR);
            }
            
            // 保存到颜色数组和原始颜色数组
            colors[i * 3] = color.r;
            colors[i * 3 + 1] = color.g;
            colors[i * 3 + 2] = color.b;
            
            originalColors[i * 3] = color.r;
            originalColors[i * 3 + 1] = color.g;
            originalColors[i * 3 + 2] = color.b;
        }
    }

    // 2. 生成爆炸形态数据
    function generateExplosion() {
        for (let i = 0; i < PARTICLE_COUNT; i++) {
            // 球形随机分布
            const theta = Math.random() * Math.PI * 2;
            const phi = Math.acos((Math.random() * 2) - 1);
            const r = 20 + Math.random() * EXPLOSION_RADIUS;

            posExplode[i * 3] = r * Math.sin(phi) * Math.cos(theta);
            posExplode[i * 3 + 1] = r * Math.sin(phi) * Math.sin(theta);
            posExplode[i * 3 + 2] = r * Math.cos(phi);
        }
    }

    // 初始化粒子位置数据
    generateHouse();
    generateExplosion();
    initDancers(); // 初始化跳舞小人

    // --- 构建 Geometry ---
    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(posCurrent, 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3)); // 添加颜色属性

    const material = new THREE.PointsMaterial({
        size: 0.4,
        map: createSprite(),
        transparent: true,
        opacity: 0.9,
        blending: THREE.AdditiveBlending,
        depthWrite: false,
        vertexColors: true // 启用顶点颜色（关键！）
    });

    const particles = new THREE.Points(geometry, material);
    scene.add(particles);

    // --- 背景星空 ---
    const starGeo = new THREE.BufferGeometry();
    const starCount = 2000;
    const starPos = new Float32Array(starCount * 3);
    for(let i=0; i<starCount*3; i++) starPos[i] = (Math.random() - 0.5) * 200;
    starGeo.setAttribute('position', new THREE.BufferAttribute(starPos, 3));
    const starMat = new THREE.PointsMaterial({size: 0.5, color: 0x555555});
    const starSystem = new THREE.Points(starGeo, starMat);
    scene.add(starSystem);

    // --- 后处理 (Bloom 辉光) ---
    const renderScene = new THREE.RenderPass(scene, camera);
    const bloomPass = new THREE.UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
    bloomPass.threshold = 0;
    bloomPass.strength = 1.2; // 辉光强度
    bloomPass.radius = 0.5;

    const composer = new THREE.EffectComposer(renderer);
    composer.addPass(renderScene);
    composer.addPass(bloomPass);

    // --- 比耶手势检测函数 ---
    function detectPeaceSign(landmarks) {
        // 获取关键手指的坐标
        const indexTip = landmarks[8];    // 食指尖
        const indexPIP = landmarks[6];    // 食指中间关节
        const middleTip = landmarks[12];  // 中指尖
        const middlePIP = landmarks[10];  // 中指中间关节
        const ringTip = landmarks[16];    // 无名指尖
        const pinkyTip = landmarks[20];   // 小拇指尖
        const thumbTip = landmarks[4];    // 大拇指尖
        
        // 计算食指和中指是否伸直（比耶的两根手指）
        const indexExtended = (indexTip.y < indexPIP.y - 0.05);
        const middleExtended = (middleTip.y < middlePIP.y - 0.05);
        
        // 计算其他手指是否弯曲
        const ringBent = (ringTip.y > landmarks[14].y);
        const pinkyBent = (pinkyTip.y > landmarks[18].y);
        const thumbBent = (Math.abs(thumbTip.x - indexTip.x) > 0.1);
        
        // 比耶手势判定条件
        return indexExtended && middleExtended && ringBent && pinkyBent && thumbBent;
    }

    // --- MediaPipe 手势逻辑 ---
    const videoElement = document.getElementById('input_video');
    
    function onResults(results) {
        document.getElementById('loading').style.display = 'none';

        if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
            const landmarks = results.multiHandLandmarks[0];
            
            // 检测比耶手势
            const isPeaceSign = detectPeaceSign(landmarks);
            
            if (isPeaceSign) {
                // 比耶手势 -> 跳舞小人
                currentState = STATE_DANCING;
                document.getElementById('video-container').style.borderColor = "#ffff00";
                showDancers = true;
            } else {
                // 计算开合度：指尖到掌根的距离
                const wrist = landmarks[0];
                const fingerTip = landmarks[8]; // 食指尖
                const middleTip = landmarks[12]; // 中指尖

                // 计算综合距离（增加中指提高准确性）
                const distance1 = Math.sqrt(
                    Math.pow(fingerTip.x - wrist.x, 2) + 
                    Math.pow(fingerTip.y - wrist.y, 2)
                );
                const distance2 = Math.sqrt(
                    Math.pow(middleTip.x - wrist.x, 2) + 
                    Math.pow(middleTip.y - wrist.y, 2)
                );
                const distance = (distance1 + distance2) / 2;

                // 调整手势判断逻辑：仅张开手掌触发爆炸，其他都保持房子
                if (distance > 0.25) {
                    // 手掌张开 -> 爆炸
                    currentState = STATE_EXPLODE;
                    document.getElementById('video-container').style.borderColor = "#00ff00";
                    showDancers = false;
                } else {
                    // 握拳/半握拳 -> 回到房子
                    currentState = STATE_HOUSE; 
                    document.getElementById('video-container').style.borderColor = "#ff69b4";
                    showDancers = false;
                }
            }
        } else {
            // 无手势 -> 回到房子
            currentState = STATE_HOUSE;
            document.getElementById('video-container').style.borderColor = "#ff69b4";
            showDancers = false;
        }
        
        // 显示/隐藏跳舞小人
        if (danceGroup) {
            danceGroup.visible = showDancers;
        }
        
        // 记录状态变化
        previousState = currentState;
    }

    // 初始化MediaPipe Hands
    const hands = new Hands({locateFile: (file) => {
        return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
    }});
    
    hands.setOptions({
        maxNumHands: 1,
        modelComplexity: 1,
        minDetectionConfidence: 0.5,
        minTrackingConfidence: 0.5
    });
    hands.onResults(onResults);

    // 初始化摄像头
    async function initCamera() {
        try {
            const cameraUtils = new Camera(videoElement, {
                onFrame: async () => {
                    await hands.send({image: videoElement});
                },
                width: 320,
                height: 240
            });
            await cameraUtils.start();
        } catch (error) {
            console.error("摄像头初始化失败:", error);
            document.getElementById('loading').innerHTML = "摄像头初始化失败<br>请检查权限或设备";
        }
    }

    // 启动摄像头
    initCamera();

    // --- 动画循环 ---
    const clock = new THREE.Clock();

    function animate() {
        requestAnimationFrame(animate);
        
        const time = clock.getElapsedTime();
        const positions = particles.geometry.attributes.position.array;
        const colorArray = particles.geometry.attributes.color.array;
        
        // 动态速度因子：爆炸时稍慢，回归时稍快
        let speed = TRANSITION_SPEED;
        if (currentState === STATE_EXPLODE && previousState === STATE_HOUSE) {
            speed = 0.08; // 爆炸时慢一点，效果更舒展
        } else if (currentState === STATE_HOUSE && previousState === STATE_EXPLODE) {
            speed = 0.15; // 回归时快一点，更快恢复房子形态
        }

        // 更新粒子位置和颜色
        for (let i = 0; i < PARTICLE_COUNT; i++) {
            const idx = i * 3;
            let tx, ty, tz;

            if (currentState === STATE_HOUSE) {
                // 目标：房子（红顶粉房+红窗）
                tx = posHouse[idx];
                ty = posHouse[idx + 1];
                tz = posHouse[idx + 2];
                
                // 房子状态下的轻微呼吸动效
                ty += Math.sin(time * 2 + tx) * 0.05; 

                // 恢复粒子的原始颜色（红顶/粉房/红窗）
                colorArray[idx] += (originalColors[idx] - colorArray[idx]) * 0.1;
                colorArray[idx + 1] += (originalColors[idx + 1] - colorArray[idx + 1]) * 0.1;
                colorArray[idx + 2] += (originalColors[idx + 2] - colorArray[idx + 2]) * 0.1;

            } else if (currentState === STATE_EXPLODE) {
                // 目标：爆炸
                tx = posExplode[idx];
                ty = posExplode[idx + 1];
                tz = posExplode[idx + 2];
                
                // 爆炸时稍微旋转
                const rotSpeed = 0.5;
                const x = tx * Math.cos(time*rotSpeed) - tz * Math.sin(time*rotSpeed);
                const z = tx * Math.sin(time*rotSpeed) + tz * Math.cos(time*rotSpeed);
                tx = x; tz = z;

                // 爆炸时统一为金色
                const gold = new THREE.Color(0xffcc00);
                colorArray[idx] = gold.r;
                colorArray[idx + 1] = gold.g;
                colorArray[idx + 2] = gold.b;
            } else if (currentState === STATE_DANCING) {
                // 跳舞状态：粒子保持房子形态，但添加欢快的动效
                tx = posHouse[idx];
                ty = posHouse[idx + 1];
                tz = posHouse[idx + 2];
                
                // 欢快的抖动效果
                const danceOffset = Math.sin(time * 8 + idx) * 0.2;
                ty += danceOffset;
                
                // 颜色变为欢快的彩色
                const hue = (time * 0.5 + idx * 0.0001) % 1;
                const danceColor = new THREE.Color().setHSL(hue, 0.8, 0.6);
                colorArray[idx] = danceColor.r;
                colorArray[idx + 1] = danceColor.g;
                colorArray[idx + 2] = danceColor.b;
            }

            // 核心动画算法：当前位置趋向目标位置 (平滑插值)
            positions[idx] += (tx - positions[idx]) * speed;
            positions[idx + 1] += (ty - positions[idx + 1]) * speed;
            positions[idx + 2] += (tz - positions[idx + 2]) * speed;
        }

        // 强制更新位置和颜色属性
        particles.geometry.attributes.position.needsUpdate = true;
        particles.geometry.attributes.color.needsUpdate = true;
        
        // 星空旋转
        starSystem.rotation.y += 0.0005;

        // --- 跳舞小人动画 ---
        if (showDancers && dancer1 && dancer2) {
            // 整体绕圈旋转
            danceGroup.rotation.y += 0.02;
            
            // 小人1跳舞动画
            dancer1.rotation.y += 0.05;
            dancer1.position.y = Math.sin(time * 3) * 0.8;
            dancer1.rotation.z = Math.sin(time * 4) * 0.2;
            
            // 手臂摆动
            const leftArm1 = dancer1.children.find(child => child.position.x < 0 && child.position.y > 2);
            const rightArm1 = dancer1.children.find(child => child.position.x > 0 && child.position.y > 2);
            if (leftArm1) leftArm1.rotation.z = Math.sin(time * 5) * 0.8;
            if (rightArm1) rightArm1.rotation.z = -Math.sin(time * 5) * 0.8;
            
            // 小人2跳舞动画
            dancer2.rotation.y -= 0.05;
            dancer2.position.y = Math.sin(time * 3 + Math.PI) * 0.8;
            dancer2.rotation.z = Math.sin(time * 4 + Math.PI) * 0.2;
            
            // 手臂摆动
            const leftArm2 = dancer2.children.find(child => child.position.x < 0 && child.position.y > 2);
            const rightArm2 = dancer2.children.find(child => child.position.x > 0 && child.position.y > 2);
            if (leftArm2) leftArm2.rotation.z = Math.sin(time * 5 + Math.PI) * 0.8;
            if (rightArm2) rightArm2.rotation.z = -Math.sin(time * 5 + Math.PI) * 0.8;
            
            // 头部轻微晃动
            const head1 = dancer1.children.find(child => child.position.y > 5);
            const head2 = dancer2.children.find(child => child.position.y > 5);
            if (head1) {
                head1.rotation.x = Math.sin(time * 6) * 0.1;
                head1.rotation.z = Math.cos(time * 6) * 0.1;
            }
            if (head2) {
                head2.rotation.x = Math.sin(time * 6 + Math.PI) * 0.1;
                head2.rotation.z = Math.cos(time * 6 + Math.PI) * 0.1;
            }
        }

        controls.update();
        composer.render(); // 使用后期处理渲染
    }

    // 窗口自适应
    window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
        composer.setSize(window.innerWidth, window.innerHeight);
    });

    // 启动动画
    animate();

    // 错误处理：防止摄像头权限被拒绝
    window.addEventListener('error', (e) => {
        if (e.message.includes('camera') || e.message.includes('permission')) {
            document.getElementById('loading').innerHTML = "无法访问摄像头<br>请在浏览器设置中允许摄像头权限";
        }
    });
</script>
</body>
</html>
