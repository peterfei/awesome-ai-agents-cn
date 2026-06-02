---
title: 用视觉大模型 + IoT 搭建产线质检 Agent
author: "@potato"
date: 2026-06
stack: YOLOv8, GPT-4o Vision, Edge Device, MQTT, Python
difficulty: 专家
---

## 场景

电子/汽车零部件产线的质量检测依赖人工目检，问题突出：

1. **人眼疲劳** — 每分钟看 30-40 件，漏检率随工作时长上升
2. **标准不统一** — 不同班次、不同质检员判定标准有差异
3. **缺陷类型多** — 划痕、凹陷、色差、焊点不良、异物，规则难穷尽
4. **数据追溯难** — 发现批量缺陷时，难定位是哪台设备、哪个时段的问题

目标是：**在产线工位部署视觉 Agent，实时检测缺陷、自动分级、追溯根因**。

---

## 实现

### 架构

```
工业相机 → 边缘计算盒 → ① YOLO 快速定位 → ② VLM 精细判定
                              ↓                    ↓
                         缺陷坐标/类型         缺陷描述/等级
                              ↓                    ↓
                         PLC 控制分拣      质量数据上报 MES
```

### 核心实现

**第一步：产线图像采集**

```python
import cv2
from mqtt_client import MQTTClient

class InspectionStation:
    def __init__(self, station_id):
        self.camera = cv2.VideoCapture(0)  # 工业相机
        self.mqtt = MQTTClient()
        self.station_id = station_id
    
    def capture_on_trigger(self):
        """PLC 触发拍照（产品到位信号）"""
        while True:
            if plc.signal_received("product_in_position"):
                ret, frame = self.camera.read()
                if ret:
                    self.inspect(frame)
    
    def inspect(self, image):
        # 保存原图用于追溯
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
        cv2.imwrite(f"/data/{self.station_id}/{timestamp}.jpg", image)
        
        # 双阶段检测
        result = self.two_stage_inspection(image)
        
        # 上报结果并控制分拣
        self.report_and_sort(result, timestamp)
```

**第二步：双阶段检测（YOLO + VLM）**

```python
from ultralytics import YOLO
import base64

class VisionAgent:
    def __init__(self):
        # 第一阶段：YOLO 快速定位缺陷区域
        self.yolo = YOLO("/models/defect_detector.pt")  # 预训练/微调
        
    def two_stage_inspection(self, image):
        # Stage 1: YOLO 检测可疑区域
        yolo_results = self.yolo(image)
        
        defects = []
        for box in yolo_results[0].boxes:
            if box.conf < 0.7:
                continue  # 置信度低，可能误报
            
            # 裁剪缺陷区域
            x1, y1, x2, y2 = map(int, box.xyxy[0])
            defect_crop = image[y1:y2, x1:x2]
            
            # Stage 2: VLM 精细判定
            vlm_result = self.vlm_judge(defect_crop, box.cls)
            
            defects.append({
                "type": vlm_result["type"],
                "severity": vlm_result["severity"],
                "confidence": float(box.conf),
                "bbox": [x1, y1, x2, y2],
                "description": vlm_result["description"]
            })
        
        return defects
    
    def vlm_judge(self, crop_image, defect_class):
        """用 GPT-4o Vision 做精细缺陷分类和等级判定"""
        
        _, buffer = cv2.imencode(".jpg", crop_image)
        b64 = base64.b64encode(buffer).decode()
        
        prompt = """
        你是一位资深质检工程师。请判断以下产品缺陷的类型和严重程度。
        
        判定标准：
        - A级（致命）：影响功能或安全，必须报废
        - B级（严重）：影响外观/性能，客户可能退货
        - C级（轻微）：轻微瑕疵，可接受或降级销售
        
        输出 JSON：
        {"type": "划痕/凹陷/色差/焊点不良/异物/其他", 
         "severity": "A/B/C", 
         "description": "具体描述"}
        """
        
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=[{
                "role": "user",
                "content": [
                    {"type": "text", "text": prompt},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}}
                ]
            }],
            response_format={"type": "json_object"}
        )
        
        return json.loads(response.choices[0].message.content)
```

**第三步：PLC 联动与分拣**

```python
def report_and_sort(self, result, timestamp):
    has_fatal = any(d["severity"] == "A" for d in result["defects"])
    has_major = any(d["severity"] == "B" for d in result["defects"])
    
    if has_fatal:
        plc.send_command("reject_to_scrap")  # NG 报废通道
        grade = "报废"
    elif has_major:
        plc.send_command("reject_to_rework")  # 返工通道
        grade = "返工"
    else:
        plc.send_command("pass")  # 良品通道
        grade = "良品"
    
    # 上报 MES
    self.mqtt.publish(f"quality/{self.station_id}", {
        "timestamp": timestamp,
        "grade": grade,
        "defects": result["defects"],
        "image_path": f"/data/{self.station_id}/{timestamp}.jpg"
    })
```

**第四步：质量追溯看板**

```python
def generate_shift_report(station_id, shift_start, shift_end):
    """生成班次质量报告"""
    
    records = db.query("""
        SELECT grade, defect_type, COUNT(*) as cnt
        FROM inspection_records
        WHERE station_id=%s AND timestamp BETWEEN %s AND %s
        GROUP BY grade, defect_type
    """, (station_id, shift_start, shift_end))
    
    total = sum(r["cnt"] for r in records)
    ok_rate = sum(r["cnt"] for r in records if r["grade"]=="良品") / total
    
    prompt = f"""
    班次质量数据：
    总检数：{total}
    良品率：{ok_rate*100:.1f}%
    缺陷分布：{records.to_markdown()}
    
    请分析：
    1. 本班次质量水平是否正常
    2. 缺陷是否有集中趋势（某类缺陷突然增加 = 设备/工艺异常）
    3. 建议排查方向（哪台设备、哪个工序）
    """
    
    return llm.predict(prompt)
```

---

## 效果

在汽车零部件产线质检场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 漏检率 | 人工 3-5% → AI 0.8% |
| 检测速度 | 人工 20 件/min → AI 45 件/min |
| 质检员工作强度 | 站岗目检 → 抽检复核+处理异常 |
| 批次缺陷追溯时间 | 平均 2h → 5min（通过时间戳和图像回溯） |
| 误判率（良品判为缺陷） | 5%（需持续优化 prompt 和阈值） |

---

## 要点

1. **边缘部署是关键** — 产线网络不稳定，推理必须在边缘设备完成（NVIDIA Jetson / 华为 Atlas），云端只负责数据汇总和模型更新。

2. **YOLO + VLM 的分工** — YOLO 负责"快"（毫秒级定位），VLM 负责"准"（理解复杂缺陷语义）。纯 VLM 太慢（秒级），纯 YOLO 对没见过的新缺陷类型泛化差。

3. **光照和角度标准化** — 工业视觉最怕环境光变化。必须固定光源、固定拍摄角度，否则同一件产品上午下午判定结果不同。

4. **持续学习闭环** — 质检员对 AI 判定有异议时，可以一键"纠正"，数据回传用于微调 YOLO 模型。这是 AI 质检系统越用越准的关键。

---

## 扩展思路

- 声纹质检：电机/轴承的异常声音检测，补充视觉盲区
- 预测性维护：缺陷率突增时，自动关联设备传感器数据，预判哪台机床需要保养
- 供应商质量联动：同一原材料批次的缺陷率统计，自动反馈给供应商
- 数字孪生：缺陷热力图映射到 3D 产品模型，直观显示问题高发区域

---

## 相关资源

- [YOLOv8 官方文档](https://docs.ultralytics.com)
- [GPT-4o Vision API](https://platform.openai.com/docs/guides/vision)
- [NVIDIA Jetson 边缘部署](https://developer.nvidia.com/embedded/jetson)
- [MQTT 工业物联网协议](https://mqtt.org)
- [工业视觉打标工具 LabelImg](https://github.com/tzutalin/labelImg)
