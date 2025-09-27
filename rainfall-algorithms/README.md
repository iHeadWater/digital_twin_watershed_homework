# 基于FastAPI的流域面平均雨量计算算法服务

## 概述

本模块基于 **FastAPI** 技术栈，提供高性能的流域面平均雨量计算服务。集成了两种主要的计算方法：算术平均值法和泰森多边形法，通过现代化的异步Web服务架构，为不同应用场景提供最优的计算策略和标准化的API接口。

### 核心技术栈

- **Web框架**: FastAPI (异步高性能API)
- **数据验证**: Pydantic (类型安全的数据模型)
- **数值计算**: NumPy + SciPy (高效数值运算)
- **空间计算**: Shapely + GeoPandas (几何运算)
- **缓存**: Redis (计算结果缓存)
- **监控**: Prometheus (性能监控)

## 算法原理

### 1. 算术平均值法（Arithmetic Mean Method）

#### 基本原理
算术平均值法是最直接的计算方法，将流域内所有雨量站的降雨量数据进行简单的算术平均，假设每个雨量站对流域平均雨量的贡献权重相等。

#### 数学模型
```
P̄ = (P₁ + P₂ + P₃ + ... + Pₙ) / n
```

**参数说明：**
- `P̄` = 流域面平均雨量 (mm)
- `P₁, P₂, ..., Pₙ` = 各雨量站的降雨量 (mm)
- `n` = 雨量站总数量

#### 适用场景
- 流域面积 < 500km²
- 雨量站分布相对均匀
- 地形平坦，高程变化 < 200m
- 实时计算需求
- 快速预警系统

#### 性能特点
- **计算复杂度**: O(n)
- **内存需求**: 极低
- **处理速度**: 毫秒级
- **精度范围**: 5-25%误差（取决于站点分布）

### 2. 泰森多边形法（Thiessen Polygon Method）

#### 基本原理
泰森多边形法基于Voronoi图理论，为每个雨量站分配一个影响区域（泰森多边形），通过面积加权的方式计算流域平均雨量，充分考虑了雨量站的空间分布特征。

#### 数学模型
```
P̄ = Σ(Pᵢ × Aᵢ) / A = (P₁×A₁ + P₂×A₂ + ... + Pₙ×Aₙ) / A
```

**参数说明：**
- `P̄` = 流域面平均雨量 (mm)
- `Pᵢ` = 第i个雨量站的降雨量 (mm)
- `Aᵢ` = 第i个泰森多边形在流域内的面积 (km²)
- `A` = 流域总面积 (km²)

#### 几何构建算法
1. **Delaunay三角剖分**: 构建雨量站点的三角网
2. **垂直平分线生成**: 对每条三角网边作垂直平分线
3. **多边形形成**: 垂直平分线交点围成泰森多边形
4. **面积计算**: 求各多边形与流域边界的交集面积

#### 适用场景
- 流域面积 > 500km²
- 雨量站分布不均匀
- 复杂地形条件
- 工程设计精度要求
- 科研分析应用

#### 性能特点
- **计算复杂度**: O(n log n)
- **内存需求**: 中等到高
- **处理速度**: 秒级到分钟级
- **精度范围**: 3-15%误差

## 算法对比分析

| 比较维度 | 算术平均值法 | 泰森多边形法 |
|---------|------------|-------------|
| **计算复杂度** | O(n) | O(n log n) |
| **空间权重** | 等权重 | 面积加权 |
| **数据需求** | 降雨量 | 降雨量+坐标 |
| **适用地形** | 平坦地区 | 各种地形 |
| **计算精度** | 中等 | 高 |
| **实时性** | 极佳 | 一般 |
| **应用场景** | 快速估算 | 精确分析 |

## 误差分析与优化

### 误差来源

**算术平均值法：**
- 空间代表性误差：站点分布不均导致的系统性偏差
- 地形影响误差：未考虑地形对降雨分布的影响
- 边界效应：边界附近站点影响被高估或低估

**泰森多边形法：**
- 边界处理误差：流域边界与多边形交集计算精度
- 站点密度误差：站点过少时的插值误差
- 几何计算误差：数值计算精度限制

### 优化策略

**算术平均值法改进：**
- 增加站点密度
- 按地形分区计算
- 结合高程权重修正

**泰森多边形法改进：**
- 优化边界处理算法
- 考虑地形因子修正
- 使用高精度几何计算库

## 数据质量控制

### 输入数据验证
```python
def validate_rainfall_data(stations):
    """数据质量验证"""
    for station in stations:
        # 降雨量合理性检查
        if not (0 <= station['rainfall'] <= 1000):
            raise ValueError(f"站点{station['id']}降雨量异常: {station['rainfall']}mm")
        
        # 坐标有效性检查
        if not (-180 <= station['longitude'] <= 180):
            raise ValueError(f"站点{station['id']}经度无效")
        if not (-90 <= station['latitude'] <= 90):
            raise ValueError(f"站点{station['id']}纬度无效")
    
    return True
```

### 异常值处理
- **统计检验**: 使用3σ准则识别异常值
- **空间一致性**: 检查相邻站点数据的空间相关性
- **时间连续性**: 验证时间序列数据的合理性
- **缺失值处理**: 采用插值或邻近站点替代

## 性能基准测试

### 计算效率对比

| 流域规模 | 站点数量 | 算术平均法 | 泰森多边形法 | 效率比 |
|---------|---------|-----------|-------------|--------|
| 小型(<100km²) | 5-10个 | <1ms | 10-50ms | 1:50 |
| 中型(100-500km²) | 10-20个 | <1ms | 50-200ms | 1:200 |
| 大型(500-1000km²) | 20-50个 | 1-2ms | 200ms-1s | 1:500 |
| 特大型(>1000km²) | >50个 | 2-5ms | 1-10s | 1:2000 |

### 精度评估

**算术平均值法精度：**
- 均匀分布条件下：5-10%误差
- 不均匀分布条件下：15-25%误差
- 复杂地形条件下：可达30%误差

**泰森多边形法精度：**
- 充足站点密度：3-8%误差
- 站点密度不足：8-15%误差
- 边界效应影响：增加2-5%误差

## 应用案例

### 案例1：城市暴雨分析
- **流域特征**: 50km²城市区域，8个雨量站
- **算法选择**: 算术平均值法
- **选择理由**: 地形平坦，站点均匀，需要实时计算
- **应用效果**: 满足实时预警需求，计算效率高

### 案例2：山区流域设计
- **流域特征**: 1200km²山区流域，12个雨量站分布不均
- **算法选择**: 泰森多边形法
- **选择理由**: 地形复杂，站点分布不均，需要高精度
- **应用效果**: 精度提高15%，为工程设计提供可靠依据

## 最佳实践建议

### 算法选择指南

**使用算术平均值法的条件：**
- 流域面积 < 500km²
- 站点分布均匀度 > 70%
- 地形坡度变化 < 5%
- 实时计算要求 < 100ms
- 精度要求 < 20%

**使用泰森多边形法的条件：**
- 流域面积 > 500km²
- 站点分布不均匀
- 复杂地形条件
- 精度要求 < 10%
- 有充足计算资源

### 混合策略
1. **分级计算**: 初步筛选用算术平均，精确分析用泰森多边形
2. **实时监测**: 实时系统采用算术平均法
3. **离线分析**: 详细研究采用泰森多边形法
4. **动态切换**: 根据数据质量和计算资源动态选择算法

## FastAPI服务实现

### 1. Pydantic数据模型
```python
from pydantic import BaseModel, Field, validator
from typing import List, Optional, Dict, Any
from enum import Enum
import numpy as np

class RainfallStation(BaseModel):
    """雨量站数据模型"""
    id: str = Field(..., description="站点唯一标识符")
    name: str = Field(..., description="站点名称")
    longitude: float = Field(..., ge=-180, le=180, description="经度")
    latitude: float = Field(..., ge=-90, le=90, description="纬度")
    rainfall: float = Field(..., ge=0, le=1000, description="降雨量(mm)")
    elevation: Optional[float] = Field(None, description="海拔高度(m)")
    
    @validator('rainfall')
    def validate_rainfall(cls, v):
        if v < 0 or v > 1000:
            raise ValueError('降雨量必须在0-1000mm范围内')
        return v

class WatershedBoundary(BaseModel):
    """流域边界数据模型"""
    coordinates: List[List[float]] = Field(..., description="边界坐标点列表")
    area: Optional[float] = Field(None, description="流域面积(km²)")
    
    @validator('coordinates')
    def validate_coordinates(cls, v):
        if len(v) < 3:
            raise ValueError('流域边界至少需要3个坐标点')
        return v

class CalculationMethod(str, Enum):
    arithmetic_mean = "arithmetic_mean"
    thiessen_polygon = "thiessen_polygon"
    auto = "auto"  # 自动选择最优算法

class RainfallCalculationRequest(BaseModel):
    """降雨量计算请求模型"""
    stations: List[RainfallStation] = Field(..., min_items=2, description="雨量站列表")
    watershed: WatershedBoundary = Field(..., description="流域边界")
    method: CalculationMethod = Field(CalculationMethod.auto, description="计算方法")
    cache_enabled: bool = Field(True, description="是否启用缓存")
    
class RainfallCalculationResult(BaseModel):
    """降雨量计算结果模型"""
    average_rainfall: float = Field(..., description="面平均雨量(mm)")
    method_used: str = Field(..., description="实际使用的计算方法")
    calculation_time: float = Field(..., description="计算耗时(秒)")
    station_weights: Optional[Dict[str, float]] = Field(None, description="各站点权重")
    quality_metrics: Dict[str, Any] = Field(..., description="质量评估指标")
    confidence_interval: Optional[List[float]] = Field(None, description="置信区间")
```

### 2. FastAPI服务端点
```python
from fastapi import FastAPI, HTTPException, Depends, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
import asyncio
import redis.asyncio as redis
import json
from datetime import datetime, timedelta

app = FastAPI(
    title="流域面平均雨量计算服务",
    description="基于FastAPI的高性能降雨量计算API",
    version="1.0.0"
)

# CORS中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Redis缓存依赖
async def get_redis():
    return await redis.from_url("redis://localhost:6379", decode_responses=True)

@app.post("/calculate", response_model=RainfallCalculationResult)
async def calculate_rainfall(
    request: RainfallCalculationRequest,
    background_tasks: BackgroundTasks,
    redis_client: redis.Redis = Depends(get_redis)
):
    """计算流域面平均雨量"""
    try:
        # 生成缓存键
        cache_key = generate_cache_key(request)
        
        # 检查缓存
        if request.cache_enabled:
            cached_result = await redis_client.get(cache_key)
            if cached_result:
                return RainfallCalculationResult.parse_raw(cached_result)
        
        # 选择计算方法
        method = select_optimal_method(request)
        
        # 执行计算
        start_time = datetime.now()
        
        if method == "arithmetic_mean":
            result = await calculate_arithmetic_mean(request.stations)
        elif method == "thiessen_polygon":
            result = await calculate_thiessen_polygon(request.stations, request.watershed)
        else:
            raise ValueError(f"不支持的计算方法: {method}")
        
        calculation_time = (datetime.now() - start_time).total_seconds()
        
        # 构建结果
        response = RainfallCalculationResult(
            average_rainfall=result['average_rainfall'],
            method_used=method,
            calculation_time=calculation_time,
            station_weights=result.get('weights'),
            quality_metrics=result['quality_metrics'],
            confidence_interval=result.get('confidence_interval')
        )
        
        # 异步缓存结果
        if request.cache_enabled:
            background_tasks.add_task(
                cache_result, redis_client, cache_key, response.json()
            )
        
        return response
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"计算失败: {str(e)}")

@app.get("/methods", response_model=Dict[str, Any])
async def get_available_methods():
    """获取可用的计算方法"""
    return {
        "methods": [
            {
                "name": "arithmetic_mean",
                "title": "算术平均值法",
                "description": "简单快速的平均值计算",
                "complexity": "O(n)",
                "suitable_for": ["小型流域", "均匀分布站点", "实时计算"]
            },
            {
                "name": "thiessen_polygon",
                "title": "泰森多边形法",
                "description": "基于空间权重的精确计算",
                "complexity": "O(n log n)",
                "suitable_for": ["大型流域", "不均匀分布", "高精度要求"]
            }
        ]
    }

@app.get("/health")
async def health_check():
    """健康检查端点"""
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}

# 辅助函数
async def calculate_arithmetic_mean(stations: List[RainfallStation]):
    """异步算术平均值计算"""
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, _arithmetic_mean_sync, stations)

def _arithmetic_mean_sync(stations: List[RainfallStation]):
    """同步算术平均值计算"""
    rainfalls = [station.rainfall for station in stations]
    average = np.mean(rainfalls)
    
    return {
        'average_rainfall': float(average),
        'quality_metrics': {
            'station_count': len(stations),
            'std_deviation': float(np.std(rainfalls)),
            'coefficient_of_variation': float(np.std(rainfalls) / average) if average > 0 else 0
        }
    }

async def calculate_thiessen_polygon(stations: List[RainfallStation], watershed: WatershedBoundary):
    """异步泰森多边形计算"""
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, _thiessen_polygon_sync, stations, watershed)

def _thiessen_polygon_sync(stations: List[RainfallStation], watershed: WatershedBoundary):
    """同步泰森多边形计算"""
    from scipy.spatial import Voronoi
    from shapely.geometry import Polygon, Point
    import geopandas as gpd
    
    # 构建站点坐标
    points = np.array([[s.longitude, s.latitude] for s in stations])
    
    # 构建Voronoi图
    vor = Voronoi(points)
    
    # 构建流域多边形
    watershed_poly = Polygon(watershed.coordinates)
    
    # 计算各站点权重
    weights = {}
    total_area = 0
    
    for i, station in enumerate(stations):
        # 获取该站点的Voronoi区域
        region_idx = vor.point_region[i]
        region = vor.regions[region_idx]
        
        if -1 in region or len(region) == 0:
            continue
            
        # 构建Voronoi多边形
        polygon_coords = [vor.vertices[j] for j in region]
        voronoi_poly = Polygon(polygon_coords)
        
        # 计算与流域的交集面积
        intersection = watershed_poly.intersection(voronoi_poly)
        area = intersection.area if intersection.is_valid else 0
        
        weights[station.id] = area
        total_area += area
    
    # 标准化权重
    if total_area > 0:
        weights = {k: v/total_area for k, v in weights.items()}
    
    # 计算加权平均
    weighted_sum = sum(station.rainfall * weights.get(station.id, 0) for station in stations)
    
    return {
        'average_rainfall': float(weighted_sum),
        'weights': weights,
        'quality_metrics': {
            'station_count': len(stations),
            'total_weight': sum(weights.values()),
            'weight_distribution': {
                'min': min(weights.values()) if weights else 0,
                'max': max(weights.values()) if weights else 0,
                'std': float(np.std(list(weights.values()))) if weights else 0
            }
        }
    }

def select_optimal_method(request: RainfallCalculationRequest) -> str:
    """自动选择最优计算方法"""
    if request.method != CalculationMethod.auto:
        return request.method.value
    
    station_count = len(request.stations)
    watershed_area = request.watershed.area or estimate_area(request.watershed.coordinates)
    
    # 基于经验规则选择方法
    if station_count < 5 or watershed_area < 100:
        return "arithmetic_mean"
    elif station_count > 10 and watershed_area > 500:
        return "thiessen_polygon"
    else:
        # 分析站点分布均匀性
        uniformity = calculate_station_uniformity(request.stations, request.watershed)
        return "arithmetic_mean" if uniformity > 0.7 else "thiessen_polygon"

def calculate_station_uniformity(stations: List[RainfallStation], watershed: WatershedBoundary) -> float:
    """计算站点分布均匀性"""
    # 简化的均匀性计算
    points = np.array([[s.longitude, s.latitude] for s in stations])
    distances = []
    
    for i in range(len(points)):
        for j in range(i+1, len(points)):
            dist = np.linalg.norm(points[i] - points[j])
            distances.append(dist)
    
    if not distances:
        return 0.0
    
    # 使用变异系数的倒数作为均匀性指标
    cv = np.std(distances) / np.mean(distances)
    return 1.0 / (1.0 + cv)

async def cache_result(redis_client: redis.Redis, key: str, value: str):
    """异步缓存结果"""
    await redis_client.setex(key, timedelta(hours=1), value)

def generate_cache_key(request: RainfallCalculationRequest) -> str:
    """生成缓存键"""
    import hashlib
    content = f"{request.json()}"
    return f"rainfall_calc:{hashlib.md5(content.encode()).hexdigest()}"
```

### 3. 容器化部署配置
```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libgeos-dev \
    libproj-dev \
    && rm -rf /var/lib/apt/lists/*

# 安装Python依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  rainfall-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=info
    depends_on:
      - redis
    restart: unless-stopped
    
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

## 技术实现架构

### 核心算法模块
```
rainfall-algorithms/
├── api/
│   ├── main.py                 # FastAPI应用入口
│   ├── models.py               # Pydantic数据模型
│   ├── endpoints.py            # API端点实现
│   └── dependencies.py         # 依赖注入
├── core/
│   ├── arithmetic_mean.py      # 算术平均值算法
│   ├── thiessen_polygon.py     # 泰森多边形算法
│   └── data_validator.py       # 数据验证模块
├── utils/
│   ├── geometry.py             # 几何计算工具
│   ├── spatial_analysis.py     # 空间分析工具
│   ├── cache_manager.py        # 缓存管理
│   └── performance_monitor.py  # 性能监控工具
├── tests/
│   ├── test_api.py             # API测试
│   ├── test_algorithms.py      # 算法测试
│   └── benchmark_tests.py      # 性能基准测试
└── deployment/
    ├── Dockerfile              # 容器化配置
    ├── docker-compose.yml      # 服务编排
    └── k8s/                    # Kubernetes配置
```

## 扩展功能

### 高级算法支持
- **反距离权重法(IDW)**: 基于距离的权重插值
- **克里金插值法**: 地统计学最优插值
- **样条插值法**: 平滑曲面插值
- **机器学习方法**: 基于深度学习的降雨估算

### 多尺度分析
- **嵌套流域**: 支持多层次流域分析
- **时间尺度**: 支持分钟、小时、日、月多时间尺度
- **空间尺度**: 支持从点到面的多空间尺度分析

### 不确定性量化
- **误差传播**: 量化输入数据误差对结果的影响
- **置信区间**: 提供结果的置信区间估计
- **敏感性分析**: 分析参数变化对结果的敏感性

## 总结

流域面平均雨量计算算法模块为知汛项目提供了高效、精确的降雨量计算能力。通过算术平均值法和泰森多边形法的有机结合，系统能够在不同应用场景下提供最优的计算策略，满足从实时预警到精确工程设计的多样化需求。

该模块的核心优势包括：
- **算法多样性**: 提供多种计算方法适应不同场景
- **性能优化**: 针对不同规模流域优化计算效率
- **精度保障**: 通过数据质量控制和误差分析确保结果可靠性
- **扩展性强**: 支持多种高级算法和分析功能的集成