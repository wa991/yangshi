# yangshi
import React, { useState, useEffect, useRef, useMemo } from 'react';
import { Card, Slider, Select, Button, Table, Tag, Space, Typography, Divider } from 'antd';
import { PlayCircleOutlined, SaveOutlined, ReloadOutlined, InfoCircleOutlined } from '@anticons/antd-icons'; // Note: Using standard icons if available, otherwise fallback text

const { Title, Text } = Typography;
const { Option } = Select;

// 物理常量
const C_SPEED = 3e8; // 光速 m/s (仅用于参考，计算主要用波长)

// 颜色定义
const COLORS = [
  { label: '红光 (650nm)', value: 650, hex: '#ff4d4f' },
  { label: '橙光 (600nm)', value: 600, hex: '#fa8c16' },
  { label: '黄光 (570nm)', value: 570, hex: '#fadb14' },
  { label: '绿光 (532nm)', value: 532, hex: '#52c41a' },
  { label: '青光 (490nm)', value: 490, hex: '#13c2c2' },
  { label: '蓝光 (470nm)', value: 470, hex: '#2f54eb' },
  { label: '紫光 (400nm)', value: 400, hex: '#722ed1' },
  { label: '白光 (混合)', value: 'white', hex: '#ffffff' },
];

const YoungsDoubleSlitSim = () => {
  // --- 状态管理 ---
  
  // 实验参数
  const [wavelength, setWavelength] = useState(650); // nm
  const [slitDistance, setSlitDistance] = useState(0.5); // mm (双缝间距 d)
  const [slitWidth, setSlitWidth] = useState(0.1); // mm (单缝宽度 a, 影响包络，简化模型中主要影响亮度分布)
  const [screenDistance, setScreenDistance] = useState(1000); // mm (双缝到屏幕距离 L)
  
  // 实验记录
  const [records, setRecords] = useState([]);
  
  // Canvas 引用
  const canvasRef = useRef(null);
  const topViewCanvasRef = useRef(null);

  // --- 辅助函数 ---

  // 记录数据
  const handleRecord = () => {
    const newRecord = {
      key: Date.now(),
      wavelength: wavelength === 'white' ? '白光' : `${wavelength} nm`,
      slitDist: `${slitDistance} mm`,
      screenDist: `${screenDistance} mm`,
      timestamp: new Date().toLocaleTimeString(),
    };
    setRecords([newRecord, ...records]);
  };

  const clearRecords = () => {
    setRecords([]);
  };

  // --- 渲染逻辑 ---

  // 1. 绘制顶部俯视光路图 (Top View)
  useEffect(() => {
    const canvas = topViewCanvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const width = canvas.width;
    const height = canvas.height;

    ctx.clearRect(0, 0, width, height);

    // 比例尺：将物理尺寸映射到像素
    // 假设画布宽度代表 2000mm (2米)
    const scaleX = width / 2000; 
    const centerY = height / 2;

    // 绘制激光器 (左侧)
    ctx.fillStyle = '#333';
    ctx.fillRect(10, centerY - 10, 40, 20);
    ctx.fillStyle = '#fff';
    ctx.font = '10px Arial';
    ctx.fillText('激光源', 15, centerY + 4);

    // 绘制扩束镜 (简单示意)
    ctx.beginPath();
    ctx.moveTo(60, centerY - 15);
    ctx.lineTo(60, centerY + 15);
    ctx.lineTo(70, centerY);
    ctx.closePath();
    ctx.fillStyle = '#ccc';
    ctx.fill();
    ctx.stroke();

    // 绘制双缝挡板
    const barrierX = 150;
    ctx.fillStyle = '#555';
    // 上部分
    ctx.fillRect(barrierX, 0, 5, centerY - 20);
    // 下部分
    ctx.fillRect(barrierX, centerY + 20, 5, height);
    // 中间缝隙 (视觉示意，非精确比例)
    ctx.fillStyle = '#000'; // 背景色遮挡
    ctx.fillRect(barrierX, centerY - 20, 5, 40);
    
    // 绘制两条缝
    ctx.fillStyle = wavelength === 'white' ? '#fff' : COLORS.find(c => c.value === wavelength)?.hex || '#f00';
    const gapVisual = Math.max(2, slitDistance * 10); // 视觉放大以便观察
    ctx.fillRect(barrierX, centerY - gapVisual/2 - 2, 5, 2);
    ctx.fillRect(barrierX, centerY + gapVisual/2, 5, 2);

    // 绘制光线 (锥形)
    ctx.beginPath();
    ctx.moveTo(70, centerY);
    ctx.lineTo(barrierX, centerY - 30);
    ctx.lineTo(barrierX, centerY + 30);
    ctx.fillStyle = 'rgba(255, 255, 255, 0.1)';
    ctx.fill();

    // 绘制光屏
    const screenX = barrierX + screenDistance * scaleX;
    ctx.fillStyle = '#ddd';
    ctx.fillRect(screenX, 0, 5, height);
    ctx.fillStyle = '#333';
    ctx.fillText('光屏', screenX + 8, centerY);

    // 标注距离 L
    ctx.beginPath();
    ctx.moveTo(barrierX + 2, centerY + 50);
    ctx.lineTo(screenX, centerY + 50);
    ctx.strokeStyle = '#666';
    ctx.stroke();
    // 箭头
    ctx.beginPath();
    ctx.moveTo(barrierX + 2, centerY + 45);
    ctx.lineTo(barrierX + 2, centerY + 55);
    ctx.moveTo(screenX, centerY + 45);
    ctx.lineTo(screenX, centerY + 55);
    ctx.stroke();
    
    ctx.fillStyle = '#666';
    ctx.fillText(`L = ${screenDistance} mm`, (barrierX + screenX) / 2 - 20, centerY + 65);

  }, [screenDistance, slitDistance, wavelength]);

  // 2. 绘制干涉条纹 (Side View / Screen View)
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const width = canvas.width;
    const height = canvas.height;

    // 清空画布
    ctx.fillStyle = '#000';
    ctx.fillRect(0, 0, width, height);

    // 物理计算参数
    // lambda (m), d (m), L (m)
    const lambdaM = (wavelength === 'white' ? 550 : wavelength) * 1e-9; 
    const dM = slitDistance * 1e-3;
    const LM = screenDistance * 1e-3;

    // 屏幕中心 y=0 对应 canvas height/2
    const centerY = height / 2;
    
    // 模拟屏幕上的点
    // 我们只计算水平方向的一维强度分布，然后垂直拉伸绘制
    
    const imageData = ctx.createImageData(width, height);
    const data = imageData.data;

    // 优化：只计算一行像素，然后复制
    const intensityRow = new Float32Array(width);
    
    // 视场范围：假设屏幕宽度对应物理宽度 20mm (可根据L动态调整，这里固定以便观察变化)
    const physicalScreenWidth = 0.02; // 20mm
    const pixelsPerMeter = width / physicalScreenWidth;

    for (let x = 0; x < width; x++) {
      // x 对应的物理位置 (相对于中心)
      const yPos = (x - width / 2) / pixelsPerMeter; 
      
      let totalR = 0, totalG = 0, totalB = 0;

      if (wavelength === 'white') {
        // 白光干涉：叠加红绿蓝三种主要成分
        const wavelengths = [650, 532, 470]; // R, G, B
        const weights = [1.0, 1.0, 1.0];
        
        wavelengths.forEach((wl, idx) => {
          const lam = wl * 1e-9;
          // 相位差 delta = d * sin(theta) approx d * y / L
          const delta = dM * yPos / LM;
          // 干涉项 cos^2(pi * delta / lambda)
          // 加上单缝衍射包络 sinc^2(pi * a * y / (lambda * L)) -- 简化处理，此处主要展示干涉
          
          const phase = (Math.PI * delta) / lam;
          const interference = Math.cos(phase) ** 2;
          
          // 简单的包络衰减，模拟能量随角度扩散
          const envelope = 1 / (1 + (yPos * 100)**2); 

          const intensity = interference * envelope;
          
          if (idx === 0) totalR += intensity * 255;
          if (idx === 1) totalG += intensity * 255;
          if (idx === 2) totalB += intensity * 255;
        });
      } else {
        // 单色光
        const colorHex = COLORS.find(c => c.value === wavelength)?.hex || '#f00';
        // 解析RGB
        const r = parseInt(colorHex.slice(1, 3), 16);
        const g = parseInt(colorHex.slice(3, 5), 16);
        const b = parseInt(colorHex.slice(5, 7), 16);

        const delta = dM * yPos / LM;
        const phase = (Math.PI * delta) / lambdaM;
        const interference = Math.cos(phase) ** 2;
        
        // 包络
        const envelope = 1 / (1 + (yPos * 50)**2);
        const intensity = interference * envelope;

        totalR = r * intensity;
        totalG = g * intensity;
        totalB = b * intensity;
      }

      intensityRow[x] = 1; // Placeholder, we store RGB separately or calculate inline below
      // 由于createImageData是RGBA，我们需要直接填充
      // 为了性能，我们在下面的循环中直接赋值
    }

    // 填充 ImageData
    for (let y = 0; y < height; y++) {
      for (let x = 0; x < width; x++) {
        const index = (y * width + x) * 4;
        
        // 重新计算当前x的强度 (为了避免存储巨大的RGB数组，这里简单复用逻辑，或者可以优化为查表)
        // 这里为了代码简洁，稍微重复计算一下，现代浏览器JS引擎很快
        const yPos = (x - width / 2) / pixelsPerMeter;
        let r = 0, g = 0, b = 0;

        if (wavelength === 'white') {
             const wavelengths = [650, 532, 470];
             wavelengths.forEach((wl, idx) => {
                const lam = wl * 1e-9;
                const delta = dM * yPos / LM;
                const phase = (Math.PI * delta) / lam;
                const interference = Math.cos(phase) ** 2;
                const envelope = 1 / (1 + (yPos * 100)**2);
                const val = interference * envelope * 255;
                if (idx === 0) r += val;
                if (idx === 1) g += val;
                if (idx === 2) b += val;
             });
        } else {
            const colorHex = COLORS.find(c => c.value === wavelength)?.hex || '#f00';
            const cr = parseInt(colorHex.slice(1, 3), 16);
            const cg = parseInt(colorHex.slice(3, 5), 16);
            const cb = parseInt(colorHex.slice(5, 7), 16);
            
            const delta = dM * yPos / LM;
            const phase = (Math.PI * delta) / lambdaM;
            const interference = Math.cos(phase) ** 2;
            const envelope = 1 / (1 + (yPos * 50)**2);
            const factor = interference * envelope;
            
            r = cr * factor;
            g = cg * factor;
            b = cb * factor;
        }

        data[index] = r;     // R
        data[index + 1] = g; // G
        data[index + 2] = b; // B
        data[index + 3] = 255; // Alpha
      }
    }

    ctx.putImageData(imageData, 0, 0);

  }, [wavelength, slitDistance, screenDistance]);

  // 表格列定义
  const columns = [
    { title: '时间', dataIndex: 'timestamp', key: 'timestamp', width: 100 },
    { title: '波长', dataIndex: 'wavelength', key: 'wavelength' },
    { title: '双缝间距 (d)', dataIndex: 'slitDist', key: 'slitDist' },
    { title: '缝屏距离 (L)', dataIndex: 'screenDist', key: 'screenDist' },
  ];

  return (
    <div style={{ padding: '20px', fontFamily: 'sans-serif', backgroundColor: '#f0f2f5', minHeight: '100vh' }}>
      <Title level={2} style={{ textAlign: 'center', marginBottom: '20px' }}>
        杨氏双缝干涉仿真实验
      </Title>

      <div style={{ display: 'flex', gap: '20px', flexWrap: 'wrap' }}>
        
        {/* 左侧：控制面板 */}
        <Card title="实验参数设置" style={{ width: 300, flexShrink: 0 }} bodyStyle={{ padding: '15px' }}>
          <Space direction="vertical" size="large" style={{ width: '100%' }}>
            
            <div>
              <Text strong>光源波长 (λ)</Text>
              <Select 
                style={{ width: '100%', marginTop: 5 }} 
                value={wavelength} 
                onChange={setWavelength}
              >
                {COLORS.map(c => (
                  <Option key={c.value} value={c.value}>
                    <span style={{ display: 'inline-block', width: 10, height: 10, borderRadius: '50%', backgroundColor: c.hex, marginRight: 8 }}></span>
                    {c.label}
                  </Option>
                ))}
              </Select>
            </div>

            <div>
              <div style={{ display: 'flex', justifyContent: 'space-between' }}>
                <Text strong>双缝间距 (d)</Text>
                <Text type="secondary">{slitDistance} mm</Text>
              </div>
              <Slider 
                min={0.1} max={2.0} step={0.1} 
                value={slitDistance} 
                onChange={setSlitDistance} 
              />
              <Text type="secondary" style={{ fontSize: 12 }}>影响条纹间距：d 越小，条纹越宽</Text>
            </div>

            <div>
              <div style={{ display: 'flex', justifyContent: 'space-between' }}>
                <Text strong>缝屏距离 (L)</Text>
                <Text type="secondary">{screenDistance} mm</Text>
              </div>
              <Slider 
                min={500} max={2000} step={10} 
                value={screenDistance} 
                onChange={setScreenDistance} 
              />
               <Text type="secondary" style={{ fontSize: 12 }}>影响条纹间距：L 越大，条纹越宽</Text>
            </div>

            <Divider />

            <div style={{ display: 'flex', gap: 10 }}>
              <Button type="primary" icon={<SaveOutlined />} onClick={handleRecord} block>
                记录当前数据
              </Button>
              <Button icon={<ReloadOutlined />} onClick={clearRecords} danger>
                清空记录
              </Button>
            </div>
          </Space>
        </Card>

        {/* 右侧：可视化区域 */}
        <div style={{ flex: 1, display: 'flex', flexDirection: 'column', gap: '20px' }}>
          
          {/* 1. 俯视图 (光学平台布局) */}
          <Card title="光学平台俯视图 (光路示意)" bodyStyle={{ padding: 0, overflow: 'hidden' }}>
            <canvas 
              ref={topViewCanvasRef} 
              width={800} 
              height={200} 
              style={{ width: '100%', height: 'auto', display: 'block', backgroundColor: '#fafafa' }}
            />
          </Card>

          {/* 2. 干涉条纹预览 */}
          <Card title="光屏干涉图像预览" bodyStyle={{ padding: 0, overflow: 'hidden', backgroundColor: '#000' }}>
             <div style={{ position: 'relative', width: '100%', height: 300, display: 'flex', justifyContent: 'center', alignItems: 'center' }}>
                <canvas 
                  ref={canvasRef} 
                  width={800} 
                  height={300} 
                  style={{ width: '100%', height: '100%', display: 'block' }}
                />
                <div style={{ position: 'absolute', top: 10, right: 10, color: 'rgba(255,255,255,0.5)', fontSize: 12 }}>
                   Simulated Screen View
                </div>
             </div>
          </Card>

          {/* 3. 数据记录表 */}
          <Card title="实验数据记录" extra={<InfoCircleOutlined />}>
            <Table 
              dataSource={records} 
              columns={columns} 
              pagination={{ pageSize: 5 }} 
              size="small"
              locale={{ emptyText: '暂无记录，请点击“记录当前数据”' }}
            />
          </Card>

        </div>
      </div>
      
      <div style={{ marginTop: 20, color: '#888', fontSize: 12, textAlign: 'center' }}>
        <p>注：本仿真基于理想双缝干涉公式 $I = I_0 \cos^2(\frac{\pi d y}{\lambda L})$ 进行简化模拟。</p>
      </div>
    </div>
  );
};

export default YoungsDoubleSlitSim;
