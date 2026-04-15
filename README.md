import React, { useState, useEffect, useRef, useCallback } from 'react';
import * as THREE from 'three';
import { 
  Rocket, Settings2, PlaneTakeoff, BookOpen, Newspaper, Archive, 
  Wind, CloudRain, Sun, CloudLightning, Save, Play, RefreshCw, 
  AlertTriangle, FlaskConical, Wrench, Paintbrush, Award, Eye, EyeOff, 
  Sparkles, Bot, Search, Database, Camera, Image as ImageIcon, ChevronRight, 
  User, LogOut, Shield, Activity, Sliders, Lock, Terminal, Target, DollarSign, Weight,
  Info, X, Key
} from 'lucide-react';

// --- Gemini API 配置（持久化到 localStorage）---
const STORAGE_KEY_API = 'gemini_api_key';

const getApiKey = () => {
  try {
    return localStorage.getItem(STORAGE_KEY_API) || '';
  } catch {
    return '';
  }
};

const setApiKey = (key) => {
  try {
    localStorage.setItem(STORAGE_KEY_API, key);
  } catch {}
};

// --- Gemini API 封装 ---
const callGemini = async (prompt, isJson = false) => {
  const apiKey = getApiKey();
  if (!apiKey) throw new Error('请先在设置中配置 Gemini API Key');

  const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;
  const payload = { contents: [{ parts: [{ text: prompt }] }] };
  if (isJson) payload.generationConfig = { responseMimeType: "application/json" };
  const retries = [1000, 2000, 4000];
  for (let attempt = 0; attempt <= retries.length; attempt++) {
    try {
      const response = await fetch(url, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) });
      if (!response.ok) {
        const errText = await response.text().catch(() => '');
        throw new Error(`API error ${response.status}: ${errText}`);
      }
      const data = await response.json();
      const text = data.candidates?.[0]?.content?.parts?.[0]?.text;
      if (!text) throw new Error('API 返回内容为空');
      if (isJson) return JSON.parse(text.replace(/```json/g, '').replace(/```/g, ''));
      return text;
    } catch (error) {
      if (attempt === retries.length) throw error;
      await new Promise(resolve => setTimeout(resolve, retries[attempt]));
    }
  }
};

const callImagen = async (promptText) => {
  const apiKey = getApiKey();
  if (!apiKey) throw new Error('请先在设置中配置 Gemini API Key');

  const url = `https://generativelanguage.googleapis.com/v1beta/models/imagen-4.0-generate-001:predict?key=${apiKey}`;
  const payload = { instances: { prompt: promptText }, parameters: { sampleCount: 1 } };
  const retries = [1000, 2000, 4000];
  for (let attempt = 0; attempt <= retries.length; attempt++) {
    try {
      const response = await fetch(url, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) });
      if (!response.ok) throw new Error(`Imagen API error ${response.status}`);
      const result = await response.json();
      if (!result.predictions?.[0]?.bytesBase64Encoded) throw new Error('Imagen 返回格式异常');
      return `data:image/png;base64,${result.predictions[0].bytesBase64Encoded}`;
    } catch (error) {
      if (attempt === retries.length) throw error;
      await new Promise(resolve => setTimeout(resolve, retries[attempt]));
    }
  }
};

// --- 工程数据字典 ---
const MATERIALS_INFO = {
 '304L不锈钢': { name: '304L 不锈钢', costMultiplier: 0.8, strength: 95, massFactor: 1.5, color: 0xdddddd, metalness: 0.95, roughness: 0.15 },
 '2219航空铝合金': { name: '2219 航空铝合金', costMultiplier: 1.2, strength: 75, massFactor: 1.0, color: 0xd8d8d8, metalness: 0.6, roughness: 0.4 },
 '5A06防锈铝': { name: '5A06 防锈铝', costMultiplier: 1.0, strength: 70, massFactor: 1.1, color: 0xc4c4c4, metalness: 0.5, roughness: 0.5 },
 '碳纤维复合材料': { name: '碳纤维复合材料', costMultiplier: 3.5, strength: 90, massFactor: 0.4, color: 0x222222, metalness: 0.2, roughness: 0.85 },
 '钛合金': { name: '钛合金', costMultiplier: 4.0, strength: 100, massFactor: 0.9, color: 0x8a8a95, metalness: 0.8, roughness: 0.3 }
};

const BLUEPRINTS = {
 'custom': { name: '自定义轨道载具', desc: '灵活配置，应对多样化发射任务。', bgImg: 'https://images.unsplash.com/photo-1462331940025-496dfbfc7564?q=80&w=1000', baseThrust: 8000, baseMass: 300, baseCost: 20 },
 'starship': { 
   name: 'Starship (星舰)', desc: '完全可重复使用的超重型星际系统。', bgImg: 'https://images.unsplash.com/photo-1541185933-ef5d8ed016c2?q=80&w=1000',
   mat: '304L不锈钢', ox: 'LOX', fu: 'CH4', oxRatio: 3.6, color: '#cccccc', shape: 'ogive', baseThrust: 75000, baseMass: 5000, baseCost: 100,
   hardware: [ { stage: 'Super Heavy', items: ['33台 Raptor 发动机', '不锈钢储罐', '热分离环'] }, { stage: 'Starship', items: ['6台 Raptor', '隔热防护瓦', '前后控制襟翼'] } ]
 },
 'falcon9': { 
   name: 'Falcon 9 (猎鹰9号)', desc: '主力中型可重复使用火箭。', bgImg: 'https://images.unsplash.com/photo-1517976487492-5750f3195933?q=80&w=1000',
   mat: '2219航空铝合金', ox: 'LOX', fu: 'RP-1', oxRatio: 2.36, color: '#ffffff', shape: 'ogive', baseThrust: 7600, baseMass: 550, baseCost: 62,
   hardware: [ { stage: '第一级 (Booster)', items: ['9台 Merlin 1D 发动机', '展开式着陆腿', '栅格翼'] }, { stage: '第二级', items: ['1台 Merlin 真空版', '碳纤维整流罩'] } ]
 },
 'falconheavy': { 
   name: 'Falcon Heavy (猎鹰重型)', desc: '三个 Falcon 9 芯级捆绑。', bgImg: 'https://images.unsplash.com/photo-1517976547714-720226b864c1?q=80&w=1000',
   mat: '2219航空铝合金', ox: 'LOX', fu: 'RP-1', oxRatio: 2.36, color: '#ffffff', shape: 'ogive', baseThrust: 22800, baseMass: 1420, baseCost: 150,
   hardware: [ { stage: '助推与芯级', items: ['3 个并联助推器', '共 27台 Merlin 1D 发动机'] }, { stage: '上面级', items: ['Merlin 真空版', '超大型整流罩'] } ]
 },
 'lm5': {
   name: '长征五号 (胖五)', desc: '中国现役推力最大的大型运载火箭。', bgImg: 'https://images.unsplash.com/photo-1454789548928-9efd52dc4031?q=80&w=1000',
   mat: '2219航空铝合金', ox: 'LOX', fu: 'LH2', oxRatio: 5.5, color: '#ffffff', shape: 'ogive', baseThrust: 10600, baseMass: 870, baseCost: 120,
   hardware: [ { stage: '芯一级', items: ['2台 YF-77 氢氧发动机', '5米直径贮箱'] }, { stage: '助推器 (4枚)', items: ['每枚 2台 YF-100 液氧煤油发动机'] } ]
 }
};

const PROPELLANT_STANDARDS = [
 { ox: 'LOX', fu: 'CH4', ofRange: [3.5, 3.7], engine: 'Raptor', isp: 380, costFactor: 0.8 },
 { ox: 'LOX', fu: 'RP-1', ofRange: [2.3, 2.7], engine: 'Merlin / YF-100', isp: 330, costFactor: 1.0 },
 { ox: 'LOX', fu: 'LH2', ofRange: [5.4, 6.1], engine: 'YF-77 / RS-25', isp: 430, costFactor: 2.5 },
 { ox: 'N2O4', fu: 'UDMH', ofRange: [2.1, 2.6], engine: 'YF-20', isp: 289, costFactor: 1.2 },
 { ox: 'LOX', fu: 'Ethanol', ofRange: [1.2, 1.5], engine: 'A-4', isp: 265, costFactor: 0.5 },
 { ox: 'LF2', fu: 'LH2', ofRange: [4.0, 5.0], engine: 'Theoretical', isp: 469, costFactor: 10.0 }
];

const OXIDIZERS = { 'LOX': { name: '液氧', type: '深冷', c: 0x60a5fa }, 'N2O4': { name: '四氧化二氮', type: '剧毒', c: 0xfca5a5 }, 'LF2': { name: '液氟', type: '极毒', c: 0xfef08a } };
const FUELS = { 'RP-1': { name: '航空煤油', type: '常温', c: 0xf59e0b }, 'LH2': { name: '液氢', type: '深冷', c: 0xeff6ff }, 'CH4': { name: '液态甲烷', type: '深冷', c: 0xa78bfa }, 'UDMH': { name: '偏二甲肼', type: '剧毒', c: 0xfde047 }, 'Ethanol': { name: '乙醇', type: '常温', c: 0xbae6fd } };

const LAUNCH_SITES = [
 { id: 'kennedy', name: 'KSC 39A (肯尼迪)', lat: 28.57, envMultiplier: 1.1, bgImg: 'https://images.unsplash.com/photo-1517976487492-5750f3195933?q=80&w=2000' },
 { id: 'starbase', name: 'Starbase (星际基地)', lat: 25.99, envMultiplier: 1.15, bgImg: 'https://images.unsplash.com/photo-1541185933-ef5d8ed016c2?q=80&w=2000' },
 { id: 'wenchang', name: '文昌航天发射场', lat: 19.61, envMultiplier: 1.2, bgImg: 'https://images.unsplash.com/photo-1614728263694-6fa1bc5afe87?q=80&w=2000' },
 { id: 'jiuquan', name: '酒泉卫星发射中心', lat: 40.96, envMultiplier: 0.9, bgImg: 'https://images.unsplash.com/photo-1473448912268-2022ce9509d8?q=80&w=2000' }
];

const MISSIONS = [
 { id: 'starlink', name: '近地轨道卫星组网', targetPayload: 15, maxCost: 70, desc: '将多颗通信卫星送入 LEO。需要平衡成本与运力。' },
 { id: 'mars_rover', name: '火星探测器转移轨道', targetPayload: 5, maxCost: 200, desc: '需要极高的比冲(ISP)以获得足够的逃逸速度。' },
 { id: 'space_station', name: '重型空间站核心舱', targetPayload: 22, maxCost: 150, desc: '低轨道大质量载荷任务，极度考验火箭整体推力与结构强度。' }
];

const NEWS_MOCK = [
 { date: '2026-04-12', title: '星舰 V3 轨道级发射测试：再次刷新人类运载记录', source: 'SpaceX Official' },
 { date: '2026-04-09', title: '长征九号重型火箭超大直径贮箱合练成功', source: '中国航天报' },
 { date: '2026-04-05', title: '猎鹰9号完成第400次着陆回收，一枚助推器复用破25次', source: 'Aerospace Daily' },
];

// --- 3D 渲染组件 ---
const RocketPreview3D = ({ params, launchStatus = 'idle', mode = 'lab', liveTelemetry }) => {
  const mountRef = useRef(null);
  const sceneRef = useRef(null);
  const rocketGroupRef = useRef(null);
  const fireGrpRef = useRef({ main: null });
  const hullsRef = useRef([]);
  const intsRef = useRef(null);
  const pRef = useRef(params);
  const xrayRef = useRef(false);
  const modeRef = useRef(mode);
  const animFrameRef = useRef(null);
  const timeoutRef = useRef(null);

  const [xray, setXrayState] = useState(false);

  // 同步 params 到 ref，避免动画循环闭包问题
  useEffect(() => {
    pRef.current = params;
  }, [params]);

  // 同步 mode 到 ref
  useEffect(() => {
    modeRef.current = mode;
  }, [mode]);

  // X-Ray 切换
  const setXray = useCallback((val) => {
    xrayRef.current = val;
    setXrayState(val);
  }, []);

  // 重建火箭场景（内部函数，供外部调用）
  const rebuildScene = useCallback(() => {
    const rocket = rocketGroupRef.current;
    const scene = sceneRef.current;
    if (!rocket || !scene) return;

    // 清空旧网格
    while (rocket.children.length > 0) rocket.remove(rocket.children[0]);
    hullsRef.current = [];

    const p = pRef.current;
    const arch = p.architecture || 'custom';
    const bMat = new THREE.MeshStandardMaterial({
      color: MATERIALS_INFO[p.material]?.color || 0xdddddd,
      roughness: 0.3
    });
    const ptMat = new THREE.MeshStandardMaterial({ color: p.color || '#ffffff', roughness: 0.5 });

    const hull = new THREE.Group();
    rocket.add(hull);

    // 箭体
    const coreMat = arch === 'starship' || arch === 'custom' ? bMat : ptMat;
    const core = new THREE.Mesh(new THREE.CylinderGeometry(2, 2, 20, 32), coreMat);
    hull.add(core);
    hullsRef.current.push(core);

    // 整流罩（鼻锥）
    const nPts = [];
    for (let i = 0; i <= 10; i++) {
      nPts.push(new THREE.Vector2(2 * Math.sqrt(1 - Math.pow(i / 10, 1.5)), (i / 10) * 6));
    }
    const noseMat = arch === 'starship' ? bMat : ptMat;
    const nose = new THREE.Mesh(new THREE.LatheGeometry(nPts, 32), noseMat);
    nose.position.y = 13;
    hull.add(nose);
    hullsRef.current.push(nose);

    // 助推器（falconheavy / lm5）
    if (arch === 'falconheavy' || arch === 'lm5') {
      [-4, 4].forEach(x => {
        const b = new THREE.Mesh(new THREE.CylinderGeometry(1.2, 1.2, 16, 32), ptMat);
        b.position.set(x, -2, 0);
        hull.add(b);
        hullsRef.current.push(b);
      });
    }

    // 内部结构（X-Ray 可见）
    const ints = new THREE.Group();
    ints.visible = false;
    intsRef.current = ints;
    rocket.add(ints);

    const oxColor = OXIDIZERS[p.oxidizer]?.c || 0xffffff;
    const fuColor = FUELS[p.fuel]?.c || 0xffffff;
    const oxM = new THREE.Mesh(
      new THREE.CylinderGeometry(1.8, 1.8, 7, 16),
      new THREE.MeshPhysicalMaterial({ color: oxColor, transmission: 0.5, opacity: 0.8, transparent: true })
    );
    oxM.position.y = 4;
    ints.add(oxM);

    const fuM = new THREE.Mesh(
      new THREE.CylinderGeometry(1.8, 1.8, 6, 16),
      new THREE.MeshPhysicalMaterial({ color: fuColor, transmission: 0.5, opacity: 0.8, transparent: true })
    );
    fuM.position.y = -4;
    ints.add(fuM);

    // 火焰（初始隐藏）
    const fire = new THREE.Group();
    fire.position.y = -11;
    fireGrpRef.current.main = fire;
    fire.visible = false;
    rocket.add(fire);

    let fo = 0xffaa00;
    if (p.fuel === 'CH4') fo = 0x9966ff;
    else if (p.fuel === 'LH2') fo = 0x66ccff;

    const outerF = new THREE.Mesh(
      new THREE.ConeGeometry(1.6, 12, 16),
      new THREE.MeshBasicMaterial({ color: fo, transparent: true, opacity: 0.6, blending: THREE.AdditiveBlending })
    );
    outerF.position.y = -6;
    outerF.rotation.x = Math.PI;
    outerF.name = 'fO';
    fire.add(outerF);

    const coreF = new THREE.Mesh(
      new THREE.ConeGeometry(1, 8, 16),
      new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.9, blending: THREE.AdditiveBlending })
    );
    coreF.position.y = -4;
    coreF.rotation.x = Math.PI;
    coreF.name = 'fC';
    fire.add(coreF);

    // 应用当前 X-Ray 状态
    hullsRef.current.forEach(m => {
      if (m.material) {
        m.material.transparent = xrayRef.current;
        m.material.opacity = xrayRef.current ? 0.1 : 1;
        m.material.depthWrite = !xrayRef.current;
        m.material.needsUpdate = true;
      }
    });
    if (intsRef.current) intsRef.current.visible = xrayRef.current;
  }, []);

  // 初始化 Three.js 场景
  useEffect(() => {
    if (!mountRef.current) return;

    const w = mountRef.current.clientWidth || 800;
    const h = mountRef.current.clientHeight || 600;

    const scene = new THREE.Scene();
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(40, w / h, 0.1, 1000);
    camera.position.set(0, mode === 'launch' ? 15 : 5, mode === 'launch' ? 90 : 75);

    const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, powerPreference: "high-performance" });
    renderer.setSize(w, h);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    mountRef.current.appendChild(renderer.domElement);

    // 光照
    scene.add(new THREE.AmbientLight(0xffffff, 0.4));
    const dl = new THREE.DirectionalLight(0xffffff, 2.5);
    dl.position.set(30, 50, 40);
    scene.add(dl);
    const bl = new THREE.DirectionalLight(0x3b82f6, 1.5);
    bl.position.set(-30, 20, -40);
    scene.add(bl);
    const eLight = new THREE.PointLight(0xffaa00, 0, 150);
    eLight.position.set(0, -12, 0);
    scene.add(eLight);
    eLightRef = eLight; // 保存引用供动画循环使用

    // 发射台场景
    if (mode === 'launch') {
      const ground = new THREE.Mesh(
        new THREE.PlaneGeometry(500, 500),
        new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 1 })
      );
      ground.rotation.x = -Math.PI / 2;
      ground.position.y = -15;
      scene.add(ground);

      const tower = new THREE.Mesh(
        new THREE.BoxGeometry(4, 45, 4),
        new THREE.MeshStandardMaterial({ color: 0x222222 })
      );
      tower.position.set(-10, 7.5, -6);
      scene.add(tower);
    }

    // 火箭组
    const rocket = new THREE.Group();
    rocketGroupRef.current = rocket;
    scene.add(rocket);

    // 初次构建
    rebuildScene();

    // 动画循环
    let t = 0;
    const animate = () => {
      animFrameRef.current = requestAnimationFrame(animate);
      t += 0.05;

      const p = pRef.current;
      const currentMode = modeRef.current;

      // Custom 火箭缩放插值
      if (p.architecture === 'custom') {
        const targetScale = new THREE.Vector3(p.widthScale, p.heightScale, p.widthScale);
        rocket.scale.lerp(targetScale, 0.1);
      } else {
        rocket.scale.lerp(new THREE.Vector3(1, 1, 1), 0.1);
      }

      if (currentMode === 'lab' || currentMode === 'gallery') {
        rocket.rotation.y += 0.005;
        rocket.position.y = THREE.MathUtils.lerp(rocket.position.y, Math.sin(t * 0.5) * 0.5, 0.1);
        // 确保 lab 模式下火箭可见
        if (!rocket.visible) rocket.visible = true;
        if (fireGrpRef.current.main) fireGrpRef.current.main.visible = false;
        if (eLightRef) eLightRef.intensity = 0;
      } else if (currentMode === 'launch') {
        if (liveTelemetry && liveTelemetry.alt > 0 && !liveTelemetry.failed) {
          rocket.position.y = -3 + (liveTelemetry.alt * 2);
          rocket.position.x = (Math.random() - 0.5) * 0.1;
          if (fireGrpRef.current.main) {
            fireGrpRef.current.main.visible = true;
            const fo = fireGrpRef.current.main.getObjectByName('fO');
            if (fo) fo.scale.set(1 + Math.random() * 0.1, 1 + Math.random() * 0.3, 1 + Math.random() * 0.1);
          }
          if (eLightRef) eLightRef.intensity = 5;
        } else if (liveTelemetry && liveTelemetry.failed) {
          // 故障时保留可见（火箭解体），只是隐藏火焰
          if (fireGrpRef.current.main) fireGrpRef.current.main.visible = false;
          if (eLightRef) eLightRef.intensity = 0;
        } else if (launchStatus === 'countdown') {
          if (fireGrpRef.current.main) {
            fireGrpRef.current.main.visible = true;
            const fo = fireGrpRef.current.main.getObjectByName('fO');
            if (fo) fo.scale.set(0.3, 0.2, 0.3);
          }
          if (eLightRef) eLightRef.intensity = 1;
          rocket.position.y = -3;
          rocket.rotation.y = 0;
        } else {
          // idle 状态 → 重置位置和可见性
          rocket.position.y = -3;
          rocket.position.x = 0;
          rocket.rotation.y = 0;
          rocket.visible = true;
          if (fireGrpRef.current.main) fireGrpRef.current.main.visible = false;
          if (eLightRef) eLightRef.intensity = 0;
        }
      }

      renderer.render(scene, camera);
    };
    animate();

    // 响应窗口大小变化
    const handleResize = () => {
      if (!mountRef.current || !renderer || !camera) return;
      const nw = mountRef.current.clientWidth;
      const nh = mountRef.current.clientHeight;
      camera.aspect = nw / nh;
      camera.updateProjectionMatrix();
      renderer.setSize(nw, nh);
    };
    window.addEventListener('resize', handleResize);

    // 清理函数
    return () => {
      if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
      if (timeoutRef.current) clearTimeout(timeoutRef.current);
      window.removeEventListener('resize', handleResize);
      if (mountRef.current && renderer.domElement) {
        mountRef.current.removeChild(renderer.domElement);
      }
      renderer.dispose();
      // 清除引用
      sceneRef.current = null;
      rocketGroupRef.current = null;
      fireGrpRef.current.main = null;
      eLightRef = null;
    };
  }, [mode, rebuildScene]); // mode 变化时重建整个场景

  // 监听 X-Ray 和 params 变化
  useEffect(() => {
    if (!rocketGroupRef.current) return;

    // X-Ray 切换
    hullsRef.current.forEach(m => {
      if (m.material) {
        m.material.transparent = xray;
        m.material.opacity = xray ? 0.1 : 1;
        m.material.depthWrite = !xray;
        m.material.needsUpdate = true;
      }
    });
    if (intsRef.current) intsRef.current.visible = xray;
  }, [xray]);

  // 监听 params 变化（推进剂颜色、构型等）→ 重建场景
  useEffect(() => {
    if (rocketGroupRef.current) {
      rebuildScene();
    }
  }, [params, rebuildScene]);

  return (
    <div className="absolute inset-0 w-full h-full flex justify-center items-center overflow-hidden bg-transparent">
      {(mode === 'lab' || mode === 'gallery') && (
        <button
          onClick={() => setXray(!xray)}
          className={`absolute top-4 right-4 z-30 flex items-center px-4 py-2 rounded-full border text-xs tracking-widest uppercase transition ${
            xray ? 'bg-white text-black' : 'bg-black/50 text-white/50 hover:text-white'
          }`}
        >
          {xray ? <EyeOff className="w-3 h-3 mr-2" /> : <Eye className="w-3 h-3 mr-2" />}
          {xray ? 'Exit X-Ray' : 'X-Ray'}
        </button>
      )}
      <div ref={mountRef} className="w-full h-full" />
    </div>
  );
};

// 临时引用（避免闭包问题）
let eLightRef = null;

// --- 主组件 ---
export default function App() {
  const [showGuide, setShowGuide] = useState(true);
  const [activeTab, setActiveTab] = useState('lab');
  const [labStep, setLabStep] = useState(1);

  const [rocketParams, setRocketParams] = useState({
    name: 'Project Vanguard',
    architecture: 'custom',
    shape: 'ogive',
    material: '2219航空铝合金',
    color: '#ffffff',
    oxidizer: 'LOX',
    fuel: 'RP-1',
    oxRatio: 2.6,
    widthScale: 1.0,
    heightScale: 1.0
  });

  const [activeMission, setActiveMission] = useState(MISSIONS[0]);
  const [engineStats, setEngineStats] = useState({ isp: 0, stability: 0, payloadLEO: "0", cost: "0", twRatio: "0", msg: '' });

  const [aiPlannerLoading, setAiPlannerLoading] = useState(false);
  const [aiReport, setAiReport] = useState('');
  const [aiReportLoading, setAiReportLoading] = useState(false);

  const [savedRockets, setSavedRockets] = useState([]);
  const [launchSite, setLaunchSite] = useState(LAUNCH_SITES[0]);
  const [weather, setWeather] = useState({ condition: 'CLEAR', Icon: Sun, temp: 25, wind: 5 });

  const [launchStatus, setLaunchStatus] = useState('idle');
  const [launchLog, setLaunchLog] = useState([]);
  const [telemetry, setTelemetry] = useState({ alt: 0, vel: 0, apoapsis: 0, maxQ: false, failed: false, anomaly: '' });

  const [dbSearchQuery, setDbSearchQuery] = useState('');
  const [dbSearchResult, setDbSearchResult] = useState('');
  const [dbSearchLoading, setDbSearchLoading] = useState(false);
  const [generatingImage, setGeneratingImage] = useState(null);

  const [user, setUser] = useState(null);
  const [authForm, setAuthForm] = useState({ callsign: '', passcode: '' });
  const [isAuthenticating, setIsAuthenticating] = useState(false);
  const [sysSettings, setSysSettings] = useState({ telemetry: 'metric', graphics: 'ultra', aiSafety: true });

  // API Key 配置状态
  const [apiKeyInput, setApiKeyInput] = useState(getApiKey());
  const [showApiKeyModal, setShowApiKeyModal] = useState(false);

  // 发射定时器引用（防止泄漏）
  const flightTimerRef = useRef(null);
  const countdownTimerRef = useRef(null);

  // 清理所有定时器
  const clearAllTimers = useCallback(() => {
    if (flightTimerRef.current) {
      clearInterval(flightTimerRef.current);
      flightTimerRef.current = null;
    }
    if (countdownTimerRef.current) {
      clearTimeout(countdownTimerRef.current);
      countdownTimerRef.current = null;
    }
  }, []);

  // 组件卸载时清理
  useEffect(() => {
    return () => clearAllTimers();
  }, [clearAllTimers]);

  // 计算引擎统计（修复 deltaV/payload 公式）
  useEffect(() => {
    const bp = BLUEPRINTS[rocketParams.architecture];
    const mat = MATERIALS_INFO[rocketParams.material];
    const { oxidizer, fuel, oxRatio, widthScale, heightScale } = rocketParams;

    let isp = 120, stability = 98, costFactor = 1.0, msg = '';
    const combo = PROPELLANT_STANDARDS.find(s => s.ox === oxidizer && s.fu === fuel);

    if (combo) {
      const optimal = (combo.ofRange[0] + combo.ofRange[1]) / 2;
      const ratioDiff = Math.abs(oxRatio - optimal);
      isp = Math.max(100, combo.isp - (ratioDiff * 60));
      stability = Math.max(0, 98 - (ratioDiff * 35));
      costFactor = combo.costFactor;
      if (stability < 40) msg = '⚠️ 燃烧极不稳定，推力室解体风险极高。';
      else if (stability < 80) msg = '🟡 偏离最优配比，发动机效率存在损失。';
      else msg = `✅ 推进系统状态标称，匹配 ${combo.engine} 规范。`;
    } else {
      msg = '❌ 致命配置：推进剂组合工程上不可行或极易爆炸。';
    }

    const volumeScale = widthScale * widthScale * heightScale;
    const totalMass = bp.baseMass * mat.massFactor * volumeScale;
    const totalThrust = bp.baseThrust * volumeScale * (isp / 300);
    const twRatio = totalThrust / (totalMass * 9.8);

    const g0 = 9.80665;
    const dryMass = totalMass * 0.15; // 干质量 = 15% 总质量
    const mFull = totalMass;
    const mEmpty = dryMass + 100; // +100t 有效载荷
    const deltaV = isp * g0 * Math.log(mFull / mEmpty);

    const leoVelocity = 9400; // m/s
    let payloadMass = 0;
    if (deltaV > leoVelocity * 0.8) {
      const payloadCapacity = Math.min(
        (deltaV / leoVelocity) * (totalThrust / (g0 * 1000)) * 0.5,
        totalMass * 0.05 
      );
      payloadMass = Math.max(0, payloadCapacity / mat.massFactor);
    }

    const estCost = bp.baseCost * volumeScale * mat.costMultiplier * costFactor;

    setEngineStats({
      isp: Math.round(isp),
      stability: Math.round(stability),
      payloadLEO: payloadMass.toFixed(1),
      cost: estCost.toFixed(1),
      twRatio: twRatio.toFixed(2),
      msg
    });
  }, [rocketParams]);

  const applyBlueprint = (id) => {
    const bp = BLUEPRINTS[id];
    setRocketParams(p => {
      if (id === 'custom') {
        return { ...p, architecture: id, name: p.name };
      }
      return {
        ...p,
        architecture: id,
        name: bp.name,
        material: bp.mat || p.material,
        oxidizer: bp.ox || p.oxidizer,
        fuel: bp.fu || p.fuel,
        oxRatio: bp.oxRatio ?? p.oxRatio,
        color: bp.color || p.color,
        shape: bp.shape || p.shape,
        widthScale: 1.0,
        heightScale: 1.0
      };
    });
  };

  const handleAIAssistant = async () => {
    if (!getApiKey()) {
      setShowApiKeyModal(true);
      return;
    }
    setAiPlannerLoading(true);
    try {
      const prompt = `航天任务："${activeMission.name}"。目标载荷：${activeMission.targetPayload}t，预算限制：$${activeMission.maxCost}M。
      请推荐火箭配置，并详细解释为什么在这个预算和载荷下，选择该构型和燃料组合是最优解（科普性强一点）。
      可用架构: custom, falcon9, starship, lm5。材料: 304L不锈钢, 2219航空铝合金, 碳纤维复合材料。氧化剂: LOX, N2O4。燃料: RP-1, LH2, CH4。
      返回JSON：{"architecture": "...", "material": "...", "oxidizer": "...", "fuel": "...", "oxRatio": 2.5, "aiReason": "..."}`;
      const res = await callGemini(prompt, true);
      if (res.architecture) applyBlueprint(res.architecture);
      setRocketParams(p => ({
        ...p,
        material: res.material || p.material,
        oxidizer: res.oxidizer || p.oxidizer,
        fuel: res.fuel || p.fuel,
        oxRatio: res.oxRatio ?? p.oxRatio
      }));
      alert(`🤖 AI 架构师推荐方案：\n\n【配置】${BLUEPRINTS[res.architecture]?.name} + ${res.oxidizer}/${res.fuel}\n\n【深度分析】\n${String(res.aiReason)}`);
    } catch (err) {
      alert("AI 网络繁忙，请稍后重试: " + err.message);
    } finally {
      setAiPlannerLoading(false);
    }
  };

  const generateWeather = () => {
    const conditions = ['CLEAR', 'CLOUDY', 'RAIN', 'THUNDERSTORM'];
    const icons = { 'CLEAR': Sun, 'CLOUDY': CloudRain, 'RAIN': CloudRain, 'THUNDERSTORM': CloudLightning };
    const condition = conditions[Math.floor(Math.random() * conditions.length)];
    setWeather({ condition, Icon: icons[condition], temp: Math.floor(Math.random() * 40) - 5, wind: Math.floor(Math.random() * 25) });
    setLaunchStatus('idle');
    setAiReport('');
    setLaunchLog(['SYSTEM: WEATHER DATA UPDATED. PAD SECURE.']);
  };

  useEffect(() => {
    generateWeather();
  }, [launchSite]);

  const startLaunch = () => {
    if (launchStatus !== 'idle' && launchStatus !== 'success' && launchStatus !== 'failed') return;

    // 清理旧定时器
    clearAllTimers();

    // 重置遥测数据
    setTelemetry({ alt: 0, vel: 0, apoapsis: 0, maxQ: false, failed: false, anomaly: '' });
    setAiReport('');
    setLaunchLog(['[T-5] 自动发射序列启动 (AUTO SEQUENCE START)', '[T-3] 点火程序 (IGNITION SEQUENCE)', '起飞 (LIFTOFF)']);

    setLaunchStatus('countdown');

    countdownTimerRef.current = setTimeout(() => {
      setLaunchStatus('flying');

      let currentAlt = 0;
      let currentVel = 0;
      let currentApo = 0;
      let tick = 0;
      let activeTw = parseFloat(engineStats.twRatio);
      let failed = false;

      flightTimerRef.current = setInterval(() => {
        tick++;

        // 随机故障模拟
        if (tick === 30 && Math.random() > 0.85) {
          activeTw *= 0.85;
          setTelemetry(prev => ({ ...prev, anomaly: 'ENGINE OUT DETECTED' }));
          setLaunchLog(prev => [...prev, 'WARNING: 侦测到单台发动机失效 (ENGINE OUT)。飞控正在补偿推力...']);
        }

        currentAlt += (0.5 * tick) + (activeTw > 1 ? (activeTw - 1) : 0);
        currentVel += 0.2 * tick * (activeTw / 2);
        currentApo = currentAlt * 1.8 + (currentVel * 2.5);

        setTelemetry(prev => ({
          ...prev,
          alt: currentAlt,
          vel: currentVel,
          apoapsis: currentApo,
          maxQ: currentAlt > 10 && currentAlt < 25
        }));

        // 故障判定逻辑
        if (tick === 5 && activeTw < 1 && !failed) {
          failed = true;
          clearInterval(flightTimerRef.current);
          flightTimerRef.current = null;
          setLaunchStatus('failed');
          setTelemetry(prev => ({ ...prev, failed: true }));
          setLaunchLog(prev => [...prev, 'FATAL: T/W RATIO < 1. 推力不足以克服地心引力.', '载具坠毁在发射台。']);
          return;
        }
        if (tick === 15 && engineStats.stability < 60 && !failed) {
          failed = true;
          clearInterval(flightTimerRef.current);
          flightTimerRef.current = null;
          setLaunchStatus('failed');
          setTelemetry(prev => ({ ...prev, failed: true }));
          setLaunchLog(prev => [...prev, 'FATAL: 燃烧室高频共振失稳。发动机爆炸。']);
          return;
        }
        if (tick === 25 && currentAlt > 15 && MATERIALS_INFO[rocketParams.material].strength < 80 && weather.wind > 15 && !failed) {
          failed = true;
          clearInterval(flightTimerRef.current);
          flightTimerRef.current = null;
          setLaunchStatus('failed');
          setTelemetry(prev => ({ ...prev, failed: true }));
          setLaunchLog(prev => [...prev, 'FATAL: MAX-Q 动压过大，遭遇高空风切变。结构空中解体。']);
          return;
        }

        // 入轨成功
        if (tick > 60) {
          clearInterval(flightTimerRef.current);
          flightTimerRef.current = null;
          setLaunchStatus('success');
          setLaunchLog(prev => [...prev, 'MECO (主机关机).', `轨道插入确认: ${Math.round(currentApo)}km x ${Math.round(currentAlt)}km`, '有效载荷分离成功。']);
        }
      }, 100);
    }, 3000);
  };

  const handleGenerateReport = async () => {
    if (!getApiKey()) {
      setShowApiKeyModal(true);
      return;
    }
    setAiReportLoading(true);
    try {
      const prompt = `你是飞控总监。发射记录：火箭[${rocketParams.name}], 架构[${BLUEPRINTS[rocketParams.architecture].name}], T/W推重比[${engineStats.twRatio}], 天气[${launchSite.name}, ${weather.wind}m/s], 结果[${launchStatus === 'success' ? '成功入轨' : '坠毁'}], 遥测末条[${launchLog[launchLog.length - 1]}]。写一份《飞行评估简报》(150字)。分析核心物理因素。直接输出纯文本。`;
      const res = await callGemini(prompt);
      setAiReport(res);
    } catch (err) {
      setAiReport("链路中断。");
    } finally {
      setAiReportLoading(false);
    }
  };

  const handleDatabaseSearch = async () => {
    if (!dbSearchQuery.trim()) return;
    if (!getApiKey()) {
      setShowApiKeyModal(true);
      return;
    }
    setDbSearchLoading(true);
    setDbSearchResult('');
    try {
      const prompt = `作为星际指挥中心的 AI，解答："${dbSearchQuery}"。用极致专业、大气的科技感语言，不要用 Markdown 符号，段落分明。`;
      const result = await callGemini(prompt, false);
      setDbSearchResult(result);
    } catch (err) {
      setDbSearchResult("⚠️ 数据库离线。");
    } finally {
      setDbSearchLoading(false);
    }
  };

  const handleGenerateRealImage = async (rocketId) => {
    if (!getApiKey()) {
      setShowApiKeyModal(true);
      return;
    }
    const rocket = savedRockets.find(r => r.id === rocketId);
    if (!rocket) return;
    setGeneratingImage(rocketId);
    try {
      const archName = BLUEPRINTS[rocket.architecture].name;
      const prompt = `A breathtaking, highly detailed photorealistic 8k image of a massive space rocket named "${rocket.name}". It looks like ${archName}. The rocket is painted ${rocket.color} with a ${rocket.shape} nose cone, standing majestically on a futuristic launchpad at night. Cinematic lighting, dramatic smoke, epic scale, SpaceX engineering style, sharp focus, masterpiece.`;
      const base64Img = await callImagen(prompt);
      setSavedRockets(prev => prev.map(r => r.id === rocketId ? { ...r, imageUrl: base64Img } : r));
    } catch (err) {
      alert("实景图像生成失败: " + err.message);
    } finally {
      setGeneratingImage(null);
    }
  };

  const handleSaveApiKey = () => {
    setApiKey(apiKeyInput.trim());
    setShowApiKeyModal(false);
  };

  const handleAuth = () => {
    if (!authForm.callsign.trim() || !authForm.passcode.trim()) return alert("访问拒绝：请输入完整的指挥官代号与安全密钥。");
    setIsAuthenticating(true);
    setTimeout(() => {
      setUser({ callsign: authForm.callsign.toUpperCase(), rank: '首席星际架构师', clearance: 'OMEGA LEVEL 9', joined: '2026-04-14' });
      setIsAuthenticating(false);
    }, 1500);
  };

  // --- 渲染组件 ---

  const renderApiKeyModal = () => (
    <div className="fixed inset-0 z-[200] flex items-center justify-center bg-black/80 backdrop-blur-md">
      <div className="bg-[#111] border border-white/20 p-8 rounded-3xl max-w-md w-full m-4 shadow-2xl">
        <div className="flex justify-center mb-6"><Key className="w-12 h-12 text-blue-400" /></div>
        <h2 className="text-2xl font-bold text-center text-white mb-2 tracking-tighter">配置 Gemini API Key</h2>
        <p className="text-center text-white/50 text-sm mb-6">AI 架构师、AI 报告和实景图像生成功能需要有效的 API Key。</p>
        <div className="mb-4">
          <input
            type="password"
            value={apiKeyInput}
            onChange={e => setApiKeyInput(e.target.value)}
            onKeyDown={e => e.key === 'Enter' && handleSaveApiKey()}
            placeholder="输入你的 Gemini API Key..."
            className="w-full bg-[#000] border border-white/10 rounded-xl py-3 px-4 text-white font-mono text-sm focus:outline-none focus:border-blue-500 transition"
          />
        </div>
        <button onClick={handleSaveApiKey} className="w-full py-3 rounded-xl bg-white text-black font-bold uppercase tracking-widest hover:bg-gray-200 transition text-sm">
          保存并继续
        </button>
        <button onClick={() => setShowApiKeyModal(false)} className="w-full mt-2 py-3 rounded-xl border border-white/10 text-white/50 hover:text-white transition text-sm">
          取消
        </button>
      </div>
    </div>
  );

  const renderNav = () => (
    <nav className="bg-[#000000]/80 backdrop-blur-xl border-b border-white/10 h-14 sticky top-0 z-50 flex items-center justify-center">
      <div className="max-w-[1400px] w-full flex justify-between items-center px-6">
        <div className="flex items-center space-x-2 text-white font-bold text-lg tracking-tight">
          <Rocket className="h-5 w-5" /><span>星际指挥中心</span>
        </div>
        <div className="flex space-x-6 text-xs font-semibold tracking-widest uppercase text-white/70 overflow-x-auto no-scrollbar">
          {[
            { id: 'lab', label: '总装车间' },
            { id: 'launch', label: '发射中心' },
            { id: 'gallery', label: '机库阵列' },
            { id: 'news', label: '航天智库' },
            { id: 'profile', label: '指挥官终端' }
          ].map(tab => (
            <button
              key={tab.id}
              onClick={() => setActiveTab(tab.id)}
              className={`h-14 flex items-center transition-colors hover:text-white whitespace-nowrap px-2 ${
                activeTab === tab.id ? 'text-white border-b-2 border-white' : ''
              }`}
            >
              {tab.id === 'profile' && <User className="w-4 h-4 mr-1.5" />}
              {tab.id === 'news' && <Database className="w-4 h-4 mr-1.5" />}
              {tab.label}
            </button>
          ))}
          <button
            onClick={() => setShowApiKeyModal(true)}
            className={`h-14 flex items-center transition-colors hover:text-white whitespace-nowrap px-2 ${
              getApiKey() ? 'text-emerald-400' : 'text-red-400'
            }`}
            title={getApiKey() ? 'API Key 已配置' : 'API Key 未配置'}
          >
            <Key className="w-4 h-4" />
          </button>
        </div>
      </div>
    </nav>
  );

  const renderLab = () => {
    // 处理提取渲染状态类的逻辑以防止属性内的模板字符串被误解析
    const payloadColorClass = activeMission.targetPayload > parseFloat(engineStats.payloadLEO) ? 'text-red-400' : 'text-emerald-400';
    const costColorClass = parseFloat(engineStats.cost) > activeMission.maxCost ? 'text-red-400' : 'text-white';
    const twRatioColorClass = 1 > parseFloat(engineStats.twRatio) ? 'text-red-500' : 'text-white';
    const stabilityColorClass = 40 > engineStats.stability ? 'text-red-500' : 'text-white';

    return (
      <div className="flex flex-col lg:flex-row min-h-[calc(100vh-3.5rem)] md:-m-8 -m-4">
        <div className="w-full lg:w-1/2 h-[45vh] lg:h-[calc(100vh-3.5rem)] lg:sticky top-14 bg-black border-r border-white/10 relative shrink-0">
          <RocketPreview3D params={rocketParams} mode="lab" />
          <div className="absolute bottom-6 left-6 z-30 pointer-events-none">
            <h2 className="text-3xl lg:text-4xl font-bold tracking-tighter text-white uppercase">{rocketParams.name}</h2>
            <p className="text-white/50 tracking-widest text-xs mt-1 uppercase">
              EST. LAUNCH COST: <span className="text-emerald-400 font-mono">${engineStats.cost}M</span>
            </p>
          </div>
        </div>

        <div className="w-full lg:w-1/2 bg-[#050505] overflow-y-auto px-6 py-12 lg:px-16 flex-grow">
          <div className="max-w-xl mx-auto space-y-16">
            <div>
              <h2 className="text-xl font-bold tracking-tight text-white mb-4 border-b border-white/10 pb-2 flex items-center">
                <Target className="w-5 h-5 mr-2 text-blue-500" /> 任务简报 (Mission Contracts)
              </h2>
              <select
                value={activeMission.id}
                onChange={e => setActiveMission(MISSIONS.find(m => m.id === e.target.value))}
                className="w-full bg-[#111] border border-white/10 rounded-xl p-4 text-white focus:outline-none mb-4"
              >
                {MISSIONS.map(m => (<option key={m.id} value={m.id}>{m.name}</option>))}
              </select>
              <div className="bg-blue-900/10 border border-blue-500/20 p-4 rounded-xl text-sm text-blue-200/80 mb-6">
                目标载荷: <strong className="text-white">{activeMission.targetPayload}t</strong> | 预算红线: <strong className="text-white">${activeMission.maxCost}M</strong><br />{activeMission.desc}
              </div>
              <button
                onClick={handleAIAssistant}
                disabled={aiPlannerLoading}
                className="w-full flex items-center justify-center bg-white/5 hover:bg-white/10 text-white py-3 rounded-full text-sm font-bold transition"
              >
                {aiPlannerLoading ? (
                  <RefreshCw className="w-4 h-4 animate-spin mr-2" />
                ) : (
                  <Bot className="w-4 h-4 mr-2" />
                )}
                请求 AI 架构师提供满足任务的配置
              </button>
            </div>

            <div>
              <h2 className="text-xl font-bold tracking-tight text-white mb-4 border-b border-white/10 pb-2">
                1. 架构预设 (Architecture)
              </h2>
              <div className="grid grid-cols-1 sm:grid-cols-2 gap-3 mb-6">
                {Object.keys(BLUEPRINTS).map(key => (
                  <button
                    key={key}
                    onClick={() => applyBlueprint(key)}
                    className={`p-4 rounded-xl text-left transition ${
                      rocketParams.architecture === key
                        ? 'bg-white/10 border-white/30'
                        : 'bg-[#111] border-transparent hover:bg-white/5'
                    } border`}
                  >
                    <div className={`font-bold text-sm ${rocketParams.architecture === key ? 'text-white' : 'text-white/70'}`}>
                      {BLUEPRINTS[key].name}
                    </div>
                  </button>
                ))}
              </div>

              {rocketParams.architecture === 'custom' && (
                <div className="space-y-4 bg-[#111] p-4 rounded-xl border border-white/5">
                  <input
                    type="text"
                    value={rocketParams.name}
                    onChange={e => setRocketParams({ ...rocketParams, name: e.target.value })}
                    className="w-full bg-transparent border-b border-white/20 py-2 text-white font-bold focus:outline-none"
                    placeholder="Rocket Name"
                  />
                  <div>
                    <label className="text-xs text-white/50 block mb-2">直径缩放 ({rocketParams.widthScale}x)</label>
                    <input
                      type="range"
                      min="0.5"
                      max="2.0"
                      step="0.1"
                      value={rocketParams.widthScale}
                      onChange={e => setRocketParams({ ...rocketParams, widthScale: parseFloat(e.target.value) })}
                      className="w-full"
                    />
                  </div>
                  <div>
                    <label className="text-xs text-white/50 block mb-2">高度缩放 ({rocketParams.heightScale}x)</label>
                    <input
                      type="range"
                      min="0.5"
                      max="2.5"
                      step="0.1"
                      value={rocketParams.heightScale}
                      onChange={e => setRocketParams({ ...rocketParams, heightScale: parseFloat(e.target.value) })}
                      className="w-full"
                    />
                  </div>
                  <select
                    value={rocketParams.material}
                    onChange={e => setRocketParams({ ...rocketParams, material: e.target.value })}
                    className="w-full bg-black border border-white/10 rounded-lg p-2 text-white text-sm"
                  >
                    {Object.keys(MATERIALS_INFO).map(m => (<option key={m} value={m}>{MATERIALS_INFO[m].name}</option>))}
                  </select>
                </div>
              )}
            </div>

            <div>
              <h2 className="text-xl font-bold tracking-tight text-white mb-4 border-b border-white/10 pb-2">
                2. 动力系统 (Propulsion)
              </h2>
              <div className="grid grid-cols-2 gap-4 mb-4">
                <select
                  value={rocketParams.oxidizer}
                  onChange={e => setRocketParams({ ...rocketParams, oxidizer: e.target.value })}
                  className="bg-[#111] border border-white/10 rounded-xl p-3 text-white text-sm focus:outline-none"
                >
                  {Object.keys(OXIDIZERS).map(k => (<option key={k} value={k}>{OXIDIZERS[k].name}</option>))}
                </select>
                <select
                  value={rocketParams.fuel}
                  onChange={e => setRocketParams({ ...rocketParams, fuel: e.target.value })}
                  className="bg-[#111] border border-white/10 rounded-xl p-3 text-white text-sm focus:outline-none"
                >
                  {Object.keys(FUELS).map(k => (<option key={k} value={k}>{FUELS[k].name}</option>))}
                </select>
              </div>
              <div className="mb-6">
                <div className="flex justify-between text-xs text-white/50 mb-2">
                  <span>氧燃比 O/F</span>
                  <span>{rocketParams.oxRatio.toFixed(2)}</span>
                </div>
                <input
                  type="range"
                  min="1.0"
                  max="7.0"
                  step="0.1"
                  value={rocketParams.oxRatio}
                  onChange={e => setRocketParams({ ...rocketParams, oxRatio: parseFloat(e.target.value) })}
                  className="w-full"
                />
              </div>

              <div className="bg-[#0a0a0a] rounded-2xl border border-white/10 p-5 grid grid-cols-2 gap-4">
                <div>
                  <div className="text-white/40 text-[10px] uppercase mb-1 flex items-center">
                    <Weight className="w-3 h-3 mr-1" /> Payload (LEO)
                  </div>
                  <div className={`text-2xl font-light ${payloadColorClass}`}>
                    {engineStats.payloadLEO} <span className="text-xs">t</span>
                  </div>
                </div>
                <div>
                  <div className="text-white/40 text-[10px] uppercase mb-1 flex items-center">
                    <DollarSign className="w-3 h-3 mr-1" /> Est. Cost
                  </div>
                  <div className={`text-2xl font-light ${costColorClass}`}>
                    ${engineStats.cost} <span className="text-xs">M</span>
                  </div>
                </div>
                <div>
                  <div className="text-white/40 text-[10px] uppercase mb-1">T/W Ratio</div>
                  <div className={`text-xl font-light ${twRatioColorClass}`}>
                    {engineStats.twRatio}
                  </div>
                </div>
                <div>
                  <div className="text-white/40 text-[10px] uppercase mb-1">Stability</div>
                  <div className={`text-xl font-light ${stabilityColorClass}`}>
                    {engineStats.stability}%
                  </div>
                </div>
                <div className="col-span-2 text-xs text-white/50 border-t border-white/10 pt-3 mt-1">{engineStats.msg}</div>
              </div>
            </div>

            <div className="flex flex-col sm:flex-row space-y-3 sm:space-y-0 sm:space-x-3 pb-8">
              <button
                onClick={() => {
                  setSavedRockets([...savedRockets, { ...rocketParams, stats: engineStats, id: Date.now() }]);
                  alert('已入库。');
                }}
                className="flex-1 py-4 rounded-full border border-white/20 text-white text-sm font-bold uppercase tracking-widest hover:bg-white hover:text-black transition"
              >
                Save Fleet
              </button>
              <button
                onClick={() => setActiveTab('launch')}
                className="flex-1 py-4 rounded-full bg-white text-black text-sm font-bold uppercase tracking-widest hover:bg-gray-200 transition"
              >
                Launch Center
              </button>
            </div>
          </div>
        </div>
      </div>
    );
  };

  const renderLaunch = () => {
    const WeatherIcon = weather?.Icon;
    return (
      <div className="flex flex-col lg:flex-row min-h-[calc(100vh-3.5rem)] md:-m-8 -m-4 bg-black">
        <div className="w-full lg:w-2/3 h-[50vh] lg:h-[calc(100vh-3.5rem)] relative overflow-hidden bg-black border-r border-white/10 shrink-0">
          <div className="absolute inset-0 bg-cover bg-center opacity-40" style={{ backgroundImage: `url(${launchSite.bgImg})` }}></div>
          <div className="absolute inset-0 bg-gradient-to-t from-black via-transparent to-black/80"></div>
          <RocketPreview3D params={rocketParams} launchStatus={launchStatus} mode="launch" liveTelemetry={telemetry} />

          <div className="absolute top-6 left-6 z-30">
            <h2 className="text-3xl md:text-5xl font-bold tracking-tighter text-white uppercase">{rocketParams.name}</h2>
            <div className="mt-4 flex flex-wrap gap-6 text-white/80 font-mono">
              <div>
                <div className="text-[10px] text-white/50 mb-1">ALTITUDE</div>
                <div className="text-2xl">{telemetry.alt.toFixed(1)} <span className="text-sm">km</span></div>
              </div>
              <div>
                <div className="text-[10px] text-white/50 mb-1">VELOCITY</div>
                <div className="text-2xl">M {telemetry.vel.toFixed(1)}</div>
              </div>
              <div>
                <div className="text-[10px] text-emerald-500/80 mb-1">APOAPSIS (EST)</div>
                <div className="text-2xl text-emerald-400">{telemetry.apoapsis.toFixed(0)} <span className="text-sm">km</span></div>
              </div>
            </div>
            {telemetry.maxQ && <div className="text-amber-400 font-bold animate-pulse mt-2">MAX-Q: MAXIMUM DYNAMIC PRESSURE</div>}
            {telemetry.anomaly && <div className="text-red-500 font-bold animate-pulse mt-2">{telemetry.anomaly}</div>}
          </div>

          {launchStatus === 'success' && (
            <div className="absolute inset-0 flex items-center justify-center bg-black/60 backdrop-blur-sm z-50 animate-in fade-in">
              <div className="text-center max-w-md p-6">
                <div className="text-5xl font-bold text-emerald-400 mb-2">ORBIT ACHIEVED</div>
                {aiReport ? (
                  <p className="text-sm text-white/80 leading-relaxed text-left border-t border-white/20 pt-4 mt-4">{aiReport}</p>
                ) : (
                  <button
                    onClick={handleGenerateReport}
                    disabled={aiReportLoading}
                    className="mt-4 px-6 py-2 rounded-full border border-white/30 text-xs font-bold uppercase text-white hover:bg-white hover:text-black transition"
                  >
                    {aiReportLoading ? 'Analyzing...' : 'Generate AI Flight Report'}
                  </button>
                )}
              </div>
            </div>
          )}
        </div>

        <div className="w-full lg:w-1/3 bg-[#050505] p-6 lg:p-12 flex flex-col justify-between flex-grow">
          <div className="space-y-8">
            <div>
              <h3 className="text-sm font-bold text-white/50 uppercase tracking-widest border-b border-white/10 pb-2 mb-4">Site & Weather</h3>
              <select
                value={launchSite.id}
                onChange={e => setLaunchSite(LAUNCH_SITES.find(s => s.id === e.target.value))}
                className="w-full bg-[#111] rounded-lg p-3 text-white text-sm mb-4 focus:outline-none"
              >
                {LAUNCH_SITES.map(s => (<option key={s.id} value={s.id}>{s.name}</option>))}
              </select>
              <div className="flex justify-between text-white font-mono text-sm bg-[#111] p-4 rounded-lg">
                <span>Wind: {weather?.wind} m/s</span>
                <span className="flex items-center">
                  {WeatherIcon && <WeatherIcon className="w-4 h-4 mr-2" />}
                  {weather?.condition}
                </span>
              </div>
            </div>
            <div>
              <h3 className="text-sm font-bold text-white/50 uppercase tracking-widest border-b border-white/10 pb-2 mb-4">Telemetry Logs</h3>
              <div className="bg-transparent text-white/70 font-mono text-xs h-40 overflow-y-auto space-y-1">
                {launchLog.map((log, i) => (
                  <div key={i} className={log.includes('FATAL') || log.includes('WARNING') ? 'text-red-500' : 'text-emerald-400'}>
                    {log}
                  </div>
                ))}
              </div>
            </div>
          </div>

          <button
            onClick={startLaunch}
            disabled={launchStatus === 'countdown' || launchStatus === 'flying'}
            className={`mt-8 w-full py-6 rounded-full font-black text-xl tracking-tighter uppercase transition ${
              launchStatus === 'countdown' || launchStatus === 'flying'
                ? 'bg-[#111] text-white/20'
                : 'bg-white text-black hover:bg-gray-200 shadow-[0_0_20px_rgba(255,255,255,0.2)]'
            }`}
          >
            {launchStatus === 'countdown' || launchStatus === 'flying' ? 'Sequencer Active' : 'Initiate Launch'}
          </button>
        </div>
      </div>
    );
  };

  const renderGallery = () => (
    <div className="py-12 space-y-8 max-w-6xl mx-auto">
      <div className="text-center">
        <h1 className="text-4xl font-bold text-white mb-2">Fleet Archive.</h1>
        <p className="text-white/50">检视您的航天器资产。生成 8K AI 纪念影像。</p>
      </div>
      {savedRockets.length === 0 ? (
        <div className="text-center py-24 bg-[#111] rounded-3xl border border-white/5">
          <h3 className="text-xl text-white font-bold">机库空置</h3>
        </div>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
          {savedRockets.map(r => (
            <div key={r.id} className="bg-[#0a0a0a] rounded-3xl border border-white/10 overflow-hidden">
              <div className="h-64 relative bg-black">
                {r.imageUrl ? (
                  <img src={r.imageUrl} alt={r.name} className="w-full h-full object-cover" />
                ) : (
                  <RocketPreview3D params={r} mode="gallery" />
                )}
                <div className="absolute top-4 right-4 z-30">
                  {r.imageUrl ? (
                    <span className="bg-black/50 backdrop-blur text-white text-[10px] px-3 py-1 rounded-full border border-white/20">AI RENDERED</span>
                  ) : (
                    <button
                      onClick={() => handleGenerateRealImage(r.id)}
                      disabled={generatingImage === r.id}
                      className="bg-black/50 backdrop-blur hover:bg-white hover:text-black px-3 py-1 rounded-full border border-white/20 text-[10px] text-white uppercase transition"
                    >
                      {generatingImage === r.id ? 'Generating...' : '8K Render'}
                    </button>
                  )}
                </div>
              </div>
              <div className="p-6">
                <h3 className="font-bold text-white text-2xl uppercase mb-4">{r.name}</h3>
                <div className="grid grid-cols-2 gap-4 text-xs mb-6">
                  <div>
                    <span className="block text-white/30 uppercase mb-1">Payload (LEO)</span>
                    <span className="text-emerald-400 font-mono">{r.stats.payloadLEO} t</span>
                  </div>
                  <div>
                    <span className="block text-white/30 uppercase mb-1">Launch Cost</span>
                    <span className="text-white font-mono">${r.stats.cost} M</span>
                  </div>
                </div>
                <button
                  onClick={() => {
                    setRocketParams(r);
                    setActiveTab('launch');
                  }}
                  className="w-full py-3 rounded-full border border-white/20 text-white text-sm font-bold uppercase hover:bg-white hover:text-black transition"
                >
                  Load to Pad
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );

  const renderNews = () => (
    <div className="max-w-4xl mx-auto py-12 space-y-16">
      <div className="text-center space-y-6">
        <h1 className="text-5xl md:text-7xl font-bold tracking-tighter text-white">Database.</h1>
        <p className="text-xl text-white/50 font-medium">接入星辰重工智库。提出问题，获取专业解答。</p>
        <div className="relative max-w-2xl mx-auto mt-8">
          <input
            type="text"
            value={dbSearchQuery}
            onChange={e => setDbSearchQuery(e.target.value)}
            onKeyDown={e => e.key === 'Enter' && handleDatabaseSearch()}
            placeholder="Search the archive..."
            className="w-full bg-[#111] border border-white/10 rounded-full py-5 pl-8 pr-16 text-white text-lg focus:outline-none focus:border-white transition"
          />
          <button
            onClick={handleDatabaseSearch}
            disabled={dbSearchLoading}
            className="absolute right-3 top-1/2 -translate-y-1/2 bg-white text-black p-3 rounded-full hover:bg-gray-200 transition disabled:opacity-50"
          >
            {dbSearchLoading ? <RefreshCw className="h-5 w-5 animate-spin" /> : <Search className="h-5 w-5" />}
          </button>
        </div>
      </div>
      {dbSearchResult ? (
        <div className="bg-[#111] rounded-3xl p-8 md:p-12 border border-white/10 animate-in fade-in slide-in-from-bottom-4">
          <h3 className="text-xl font-bold text-white mb-6 flex items-center">
            <Bot className="w-6 h-6 mr-3 text-white/50" /> Intelligence Report
          </h3>
          <div className="text-white/80 leading-loose text-lg whitespace-pre-wrap">{dbSearchResult}</div>
        </div>
      ) : (
        <div className="space-y-6">
          <h3 className="text-sm font-semibold text-white/50 uppercase tracking-widest border-b border-white/10 pb-4">Recent Updates</h3>
          {NEWS_MOCK.map((news, idx) => (
            <div key={idx} className="group cursor-pointer py-4 border-b border-white/5 flex flex-col md:flex-row md:items-center justify-between">
              <div className="mb-2 md:mb-0">
                <h4 className="text-xl font-bold text-white group-hover:text-blue-400 transition">{news.title}</h4>
                <p className="text-sm text-white/40 mt-1">{news.source}</p>
              </div>
              <div className="flex items-center space-x-4 text-white/50 text-sm font-mono">
                <span>{news.date}</span>
                <ChevronRight className="w-5 h-5 group-hover:translate-x-2 transition-transform" />
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );

  const renderProfile = () => {
    if (!user) {
      return (
        <div className="min-h-[calc(100vh-3.5rem)] flex items-center justify-center -m-4 md:-m-8 bg-black relative overflow-hidden">
          <div className="bg-[#050505] p-10 md:p-16 rounded-3xl border border-white/10 shadow-[0_0_50px_rgba(59,130,246,0.1)] relative z-10 w-full max-w-lg">
            <div className="text-center mb-10">
              <Terminal className="w-12 h-12 text-blue-500 mx-auto mb-4" />
              <h1 className="text-3xl font-bold text-white tracking-tighter uppercase">Space Command</h1>
              <p className="text-white/40 font-mono text-sm mt-2 tracking-widest">AUTHORIZED PERSONNEL ONLY</p>
            </div>
            <div className="space-y-6">
              <div>
                <label className="text-xs font-bold text-white/50 uppercase tracking-widest block mb-2">指挥官代号 (Call Sign)</label>
                <div className="relative">
                  <User className="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-white/30" />
                  <input
                    type="text"
                    value={authForm.callsign}
                    onChange={e => setAuthForm({ ...authForm, callsign: e.target.value })}
                    className="w-full bg-[#111] border border-white/10 rounded-xl py-4 pl-12 pr-4 text-white font-mono uppercase focus:outline-none focus:border-blue-500 transition"
                    placeholder="ENTER IDENTIFIER"
                  />
                </div>
              </div>
              <div>
                <label className="text-xs font-bold text-white/50 uppercase tracking-widest block mb-2">量子安全密钥 (Passcode)</label>
                <div className="relative">
                  <Lock className="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-white/30" />
                  <input
                    type="password"
                    value={authForm.passcode}
                    onChange={e => setAuthForm({ ...authForm, passcode: e.target.value })}
                    className="w-full bg-[#111] border border-white/10 rounded-xl py-4 pl-12 pr-4 text-white font-mono focus:outline-none focus:border-blue-500 transition"
                    placeholder="••••••••"
                  />
                </div>
              </div>
              <button
                onClick={handleAuth}
                disabled={isAuthenticating}
                className="w-full py-4 mt-4 rounded-xl bg-white text-black font-bold uppercase tracking-widest hover:bg-gray-200 transition flex justify-center items-center"
              >
                {isAuthenticating ? <RefreshCw className="w-5 h-5 animate-spin" /> : '建立神经连接 (Authorize)'}
              </button>
            </div>
          </div>
        </div>
      );
    }

    return (
      <div className="max-w-5xl mx-auto py-12 space-y-12">
        <div className="flex flex-col md:flex-row items-start md:items-center justify-between mb-8">
          <div>
            <h1 className="text-5xl font-bold tracking-tighter text-white mb-2 uppercase">{user.callsign}</h1>
            <p className="text-blue-400 font-mono tracking-widest text-sm uppercase flex items-center">
              <Shield className="w-4 h-4 mr-2" /> Clearance: {user.clearance}
            </p>
          </div>
          <button
            onClick={() => setUser(null)}
            className="mt-4 md:mt-0 flex items-center px-6 py-3 rounded-full border border-red-500/30 text-red-400 hover:bg-red-500/10 transition text-sm font-bold uppercase tracking-widest"
          >
            <LogOut className="w-4 h-4 mr-2" /> 终止连接
          </button>
        </div>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
          <div className="bg-[#111] p-6 rounded-3xl border border-white/5">
            <Activity className="w-8 h-8 text-white/20 mb-4" />
            <div className="text-white/40 text-xs uppercase tracking-widest mb-1">设计服役火箭 (Fleet)</div>
            <div className="text-4xl font-light text-white">{savedRockets.length} <span className="text-sm text-white/40">台</span></div>
          </div>
          <div className="bg-[#111] p-6 rounded-3xl border border-white/5">
            <Award className="w-8 h-8 text-white/20 mb-4" />
            <div className="text-white/40 text-xs uppercase tracking-widest mb-1">当前职务级别 (Rank)</div>
            <div className="text-xl font-medium text-white">{user.rank}</div>
          </div>
          <div className="bg-[#111] p-6 rounded-3xl border border-white/5">
            <Database className="w-8 h-8 text-white/20 mb-4" />
            <div className="text-white/40 text-xs uppercase tracking-widest mb-1">系统入网时间 (Joined)</div>
            <div className="text-xl font-mono text-white">{user.joined}</div>
          </div>
        </div>

        <div>
          <h2 className="text-2xl font-bold tracking-tight text-white mb-6 border-b border-white/10 pb-4 flex items-center">
            <Sliders className="w-6 h-6 mr-3 text-white/50" /> 系统环境偏好 (System Preferences)
          </h2>
          <div className="bg-[#050505] rounded-3xl border border-white/10 p-2 overflow-hidden">
            <div className="p-6 flex flex-col sm:flex-row sm:items-center justify-between border-b border-white/5">
              <div className="mb-4 sm:mb-0">
                <div className="font-bold text-white text-lg">遥测系统单位制</div>
                <div className="text-sm text-white/40 mt-1">切换发射中心的物理计算单位。</div>
              </div>
              <div className="flex bg-[#111] p-1 rounded-xl border border-white/10">
                <button
                  onClick={() => setSysSettings({ ...sysSettings, telemetry: 'metric' })}
                  className={`px-6 py-2 rounded-lg text-sm font-bold transition ${sysSettings.telemetry === 'metric' ? 'bg-white text-black' : 'text-white/50 hover:text-white'}`}
                >
                  公制 (Metric)
                </button>
                <button
                  onClick={() => setSysSettings({ ...sysSettings, telemetry: 'imperial' })}
                  className={`px-6 py-2 rounded-lg text-sm font-bold transition ${sysSettings.telemetry === 'imperial' ? 'bg-white text-black' : 'text-white/50 hover:text-white'}`}
                >
                  英制 (Imperial)
                </button>
              </div>
            </div>
            <div className="p-6 flex flex-col sm:flex-row sm:items-center justify-between border-b border-white/5">
              <div className="mb-4 sm:mb-0">
                <div className="font-bold text-white text-lg">物理渲染引擎负载</div>
                <div className="text-sm text-white/40 mt-1">调整 3D 蓝图与发射场景的材质质量。</div>
              </div>
              <div className="flex bg-[#111] p-1 rounded-xl border border-white/10">
                <button
                  onClick={() => setSysSettings({ ...sysSettings, graphics: 'performance' })}
                  className={`px-6 py-2 rounded-lg text-sm font-bold transition ${sysSettings.graphics === 'performance' ? 'bg-white text-black' : 'text-white/50 hover:text-white'}`}
                >
                  性能优先
                </button>
                <button
                  onClick={() => setSysSettings({ ...sysSettings, graphics: 'ultra' })}
                  className={`px-6 py-2 rounded-lg text-sm font-bold transition ${sysSettings.graphics === 'ultra' ? 'bg-white text-black' : 'text-white/50 hover:text-white'}`}
                >
                  极致 (Ultra)
                </button>
              </div>
            </div>
            <div className="p-6 flex flex-col sm:flex-row sm:items-center justify-between">
              <div className="mb-4 sm:mb-0">
                <div className="font-bold text-white text-lg">AI 安全协议覆写</div>
                <div className="text-sm text-white/40 mt-1">允许在实验室调配极其危险的推进剂而不强制阻断。</div>
              </div>
              <div className="flex items-center space-x-3">
                <span className={`text-xs font-bold uppercase tracking-widest ${sysSettings.aiSafety ? 'text-emerald-400' : 'text-red-500'}`}>
                  {sysSettings.aiSafety ? '协议生效中' : '协议已覆写'}
                </span>
                <button
                  onClick={() => setSysSettings({ ...sysSettings, aiSafety: !sysSettings.aiSafety })}
                  className={`w-14 h-8 rounded-full transition-colors relative ${sysSettings.aiSafety ? 'bg-emerald-500/20 border border-emerald-500/50' : 'bg-red-500 border border-red-500'}`}
                >
                  <div className={`w-6 h-6 rounded-full bg-white absolute top-1/2 -translate-y-1/2 transition-transform ${sysSettings.aiSafety ? 'left-1' : 'translate-x-6 left-1'}`}></div>
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-black text-[#f5f5f7] font-sans selection:bg-white/30 relative">
      {showApiKeyModal && renderApiKeyModal()}

      {showGuide && (
        <div className="fixed inset-0 z-[100] flex items-center justify-center bg-black/80 backdrop-blur-md">
          <div className="bg-[#111] border border-white/20 p-8 rounded-3xl max-w-lg w-full m-4 shadow-2xl relative overflow-hidden">
            <div className="absolute top-0 right-0 w-32 h-32 bg-blue-600/20 blur-3xl rounded-full pointer-events-none"></div>
            <button onClick={() => setShowGuide(false)} className="absolute top-4 right-4 text-white/50 hover:text-white">
              <X className="w-6 h-6" />
            </button>
            <div className="flex justify-center mb-6"><Rocket className="w-12 h-12 text-white" /></div>
            <h2 className="text-3xl font-bold text-center text-white mb-2 tracking-tighter">欢迎来到星际指挥中心</h2>
            <p className="text-center text-white/50 mb-8">在这里，你将体验真正的航天工程思维与物理法则。</p>
            <div className="space-y-6">
              <div className="flex items-start">
                <Wrench className="w-6 h-6 text-blue-400 mr-4 shrink-0" />
                <div>
                  <h4 className="text-white font-bold mb-1">1. 总装车间 (Vehicle Assembly)</h4>
                  <p className="text-sm text-white/60">根据任务需求，配置火箭的结构、推重比与推进剂组合，计算运力与成本。</p>
                </div>
              </div>
              <div className="flex items-start">
                <PlaneTakeoff className="w-6 h-6 text-emerald-400 mr-4 shrink-0" />
                <div>
                  <h4 className="text-white font-bold mb-1">2. 发射中心 (Launch Control)</h4>
                  <p className="text-sm text-white/60">应对随机天气与空中故障（如单台发动机失效），见证火箭成功入轨或在 Max-Q 解体。</p>
                </div>
              </div>
              <div className="flex items-start">
                <Eye className="w-6 h-6 text-purple-400 mr-4 shrink-0" />
                <div>
                  <h4 className="text-white font-bold mb-1">3. X-Ray 与 AI 报告</h4>
                  <p className="text-sm text-white/60">利用 X-Ray 透视火箭内部构件，每次飞行结束后可生成 AI 详细飞控报告。</p>
                </div>
              </div>
            </div>
            <button
              onClick={() => setShowGuide(false)}
              className="w-full mt-8 py-4 rounded-full bg-white text-black font-bold uppercase tracking-widest hover:bg-gray-200 transition"
            >
              开始任务 (START MISSION)
            </button>
          </div>
        </div>
      )}

      {renderNav()}
      <main className="p-4 md:p-8 max-w-[1600px] mx-auto">
        {activeTab === 'lab' && renderLab()}
        {activeTab === 'launch' && renderLaunch()}
        {activeTab === 'gallery' && renderGallery()}
        {activeTab === 'news' && renderNews()}
        {activeTab === 'profile' && renderProfile()}
      </main>
    </div>
  );
}
