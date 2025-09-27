# 面平均雨量算法技术实现

## FastAPI算法服务架构

### 核心算法服务

```python
# services/rainfall_algorithms.py
from fastapi import APIRouter, HTTPException
from typing import List, Dict, Optional
import numpy as np
import pandas as pd
from dataclasses import dataclass
from scipy.spatial import Voronoi, voronoi_plot_2d
from scipy.interpolate import griddata
import asyncio

@dataclass
class RainfallStation:
    id: str
    longitude: float
    latitude: float
    rainfall: float
    elevation: Optional[float] = None
    quality_flag: Optional[str] = "A"

@dataclass
class Watershed:
    id: str
    name: str
    area: float  # km²
    boundary_points: List[tuple]  # [(lon, lat), ...]
    center_point: tuple  # (lon, lat)

router = APIRouter()

class RainfallAlgorithmService:
    """面平均雨量算法服务"""
    
    def __init__(self):
        self.algorithms = {
            "arithmetic_mean": self.arithmetic_mean,
            "thiessen_polygon": self.thiessen_polygon,
            "inverse_distance": self.inverse_distance_weighting,
            "kriging": self.kriging_interpolation
        }
    
    async def arithmetic_mean(self, stations: List[RainfallStation], 
                            watershed: Optional[Watershed] = None) -> Dict:
        """算术平均值法"""
        # 过滤质量标记为A或B的数据
        valid_stations = [s for s in stations if s.quality_flag in ["A", "B"]]
        
        if not valid_stations:
            raise HTTPException(status_code=400, detail="No valid rainfall data")
        
        rainfall_values = [station.rainfall for station in valid_stations]
        average = np.mean(rainfall_values)
        std_dev = np.std(rainfall_values)
        
        return {
            "method": "arithmetic_mean",
            "result": round(average, 2),
            "unit": "mm",
            "station_count": len(valid_stations),
            "total_stations": len(stations),
            "standard_deviation": round(std_dev, 2),
            "min_value": round(min(rainfall_values), 2),
            "max_value": round(max(rainfall_values), 2),
            "quality_summary": self._get_quality_summary(stations)
        }
    
    async def thiessen_polygon(self, stations: List[RainfallStation], 
                              watershed: Watershed) -> Dict:
        """泰森多边形法"""
        if len(stations) < 3:
            raise HTTPException(
                status_code=400, 
                detail="At least 3 stations required for Thiessen polygon"
            )
        
        # 提取站点坐标
        points = np.array([[s.longitude, s.latitude] for s in stations])
        
        # 创建Voronoi图
        vor = Voronoi(points)
        
        # 计算每个站点的权重（基于Voronoi单元面积）
        weights = await self._calculate_thiessen_weights(
            vor, stations, watershed
        )
        
        # 计算加权平均
        weighted_sum = sum(
            station.rainfall * weight 
            for station, weight in zip(stations, weights)
        )
        
        return {
            "method": "thiessen_polygon",
            "result": round(weighted_sum, 2),
            "unit": "mm",
            "station_count": len(stations),
            "weights": [round(w, 4) for w in weights],
            "watershed_area": watershed.area,
            "quality_summary": self._get_quality_summary(stations)
        }
    
    async def inverse_distance_weighting(self, stations: List[RainfallStation], 
                                       watershed: Watershed, 
                                       power: float = 2.0) -> Dict:
        """距离反比权重法"""
        center_lon, center_lat = watershed.center_point
        
        # 计算每个站点到流域中心的距离
        distances = []
        for station in stations:
            dist = self._calculate_distance(
                center_lon, center_lat,
                station.longitude, station.latitude
            )
            distances.append(dist)
        
        # 计算权重（距离的负幂次）
        weights = []
        for dist in distances:
            if dist == 0:  # 站点在流域中心
                weight = 1.0
            else:
                weight = 1.0 / (dist ** power)
            weights.append(weight)
        
        # 归一化权重
        total_weight = sum(weights)
        normalized_weights = [w / total_weight for w in weights]
        
        # 计算加权平均
        weighted_sum = sum(
            station.rainfall * weight 
            for station, weight in zip(stations, normalized_weights)
        )
        
        return {
            "method": "inverse_distance_weighting",
            "result": round(weighted_sum, 2),
            "unit": "mm",
            "station_count": len(stations),
            "power": power,
            "weights": [round(w, 4) for w in normalized_weights],
            "distances": [round(d, 2) for d in distances],
            "quality_summary": self._get_quality_summary(stations)
        }
    
    async def kriging_interpolation(self, stations: List[RainfallStation], 
                                  watershed: Watershed) -> Dict:
        """克里金插值法（简化实现）"""
        if len(stations) < 4:
            raise HTTPException(
                status_code=400, 
                detail="At least 4 stations required for Kriging"
            )
        
        # 提取坐标和降雨值
        points = np.array([[s.longitude, s.latitude] for s in stations])
        values = np.array([s.rainfall for s in stations])
        
        # 创建插值网格
        grid_x, grid_y = np.mgrid[
            min(p[0] for p in points):max(p[0] for p in points):50j,
            min(p[1] for p in points):max(p[1] for p in points):50j
        ]
        
        # 使用scipy的griddata进行插值（简化的克里金）
        grid_values = griddata(
            points, values, (grid_x, grid_y), 
            method='cubic', fill_value=0
        )
        
        # 计算流域内的平均值
        watershed_mask = self._create_watershed_mask(
            grid_x, grid_y, watershed.boundary_points
        )
        
        masked_values = grid_values[watershed_mask]
        average_rainfall = np.mean(masked_values[~np.isnan(masked_values)])
        
        return {
            "method": "kriging_interpolation",
            "result": round(average_rainfall, 2),
            "unit": "mm",
            "station_count": len(stations),
            "grid_size": "50x50",
            "interpolation_method": "cubic",
            "quality_summary": self._get_quality_summary(stations)
        }
    
    async def _calculate_thiessen_weights(self, vor: Voronoi, 
                                        stations: List[RainfallStation], 
                                        watershed: Watershed) -> List[float]:
        """计算泰森多边形权重"""
        weights = []
        total_area = 0
        
        for i, station in enumerate(stations):
            # 获取该站点的Voronoi单元
            region_index = vor.point_region[i]
            region = vor.regions[region_index]
            
            if -1 in region or len(region) == 0:
                # 无界区域，使用默认权重
                area = watershed.area / len(stations)
            else:
                # 计算多边形面积
                polygon_points = [vor.vertices[j] for j in region]
                area = self._calculate_polygon_area(polygon_points)
                
                # 裁剪到流域边界内
                clipped_area = self._clip_to_watershed(
                    polygon_points, watershed.boundary_points
                )
                area = clipped_area if clipped_area > 0 else area
            
            weights.append(area)
            total_area += area
        
        # 归一化权重
        return [w / total_area for w in weights]
    
    def _calculate_distance(self, lon1: float, lat1: float, 
                          lon2: float, lat2: float) -> float:
        """计算两点间距离（简化的球面距离）"""
        from math import radians, cos, sin, asin, sqrt
        
        # 转换为弧度
        lon1, lat1, lon2, lat2 = map(radians, [lon1, lat1, lon2, lat2])
        
        # Haversine公式
        dlon = lon2 - lon1
        dlat = lat2 - lat1
        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
        c = 2 * asin(sqrt(a))
        r = 6371  # 地球半径（公里）
        
        return c * r
    
    def _calculate_polygon_area(self, points: List[tuple]) -> float:
        """计算多边形面积（Shoelace公式）"""
        if len(points) < 3:
            return 0
        
        area = 0
        n = len(points)
        for i in range(n):
            j = (i + 1) % n
            area += points[i][0] * points[j][1]
            area -= points[j][0] * points[i][1]
        
        return abs(area) / 2
    
    def _clip_to_watershed(self, polygon_points: List[tuple], 
                          watershed_boundary: List[tuple]) -> float:
        """将多边形裁剪到流域边界内（简化实现）"""
        # 这里使用简化的实现，实际应用中可能需要更复杂的几何算法
        # 如Sutherland-Hodgman裁剪算法
        return self._calculate_polygon_area(polygon_points)
    
    def _create_watershed_mask(self, grid_x: np.ndarray, grid_y: np.ndarray, 
                             boundary_points: List[tuple]) -> np.ndarray:
        """创建流域掩码"""
        from matplotlib.path import Path
        
        # 创建流域边界路径
        path = Path(boundary_points)
        
        # 创建网格点
        points = np.column_stack((grid_x.ravel(), grid_y.ravel()))
        
        # 检查哪些点在流域内
        mask = path.contains_points(points)
        
        return mask.reshape(grid_x.shape)
    
    def _get_quality_summary(self, stations: List[RainfallStation]) -> Dict:
        """获取数据质量摘要"""
        quality_counts = {}
        for station in stations:
            flag = station.quality_flag or "Unknown"
            quality_counts[flag] = quality_counts.get(flag, 0) + 1
        
        return {
            "total_stations": len(stations),
            "quality_distribution": quality_counts,
            "valid_stations": sum(
                count for flag, count in quality_counts.items() 
                if flag in ["A", "B"]
            )
        }

# FastAPI路由定义
@router.post("/rainfall/arithmetic-mean")
async def calculate_arithmetic_mean(
    stations: List[RainfallStation],
    watershed: Optional[Watershed] = None
):
    """算术平均值法计算面平均雨量"""
    service = RainfallAlgorithmService()
    return await service.arithmetic_mean(stations, watershed)

@router.post("/rainfall/thiessen-polygon")
async def calculate_thiessen_polygon(
    stations: List[RainfallStation],
    watershed: Watershed
):
    """泰森多边形法计算面平均雨量"""
    service = RainfallAlgorithmService()
    return await service.thiessen_polygon(stations, watershed)

@router.post("/rainfall/inverse-distance")
async def calculate_inverse_distance(
    stations: List[RainfallStation],
    watershed: Watershed,
    power: float = 2.0
):
    """距离反比权重法计算面平均雨量"""
    service = RainfallAlgorithmService()
    return await service.inverse_distance_weighting(stations, watershed, power)

@router.post("/rainfall/kriging")
async def calculate_kriging(
    stations: List[RainfallStation],
    watershed: Watershed
):
    """克里金插值法计算面平均雨量"""
    service = RainfallAlgorithmService()
    return await service.kriging_interpolation(stations, watershed)

@router.get("/rainfall/algorithms")
async def list_algorithms():
    """获取可用的算法列表"""
    return {
        "algorithms": [
            {
                "name": "arithmetic_mean",
                "title": "算术平均值法",
                "description": "简单的算术平均，适用于站点分布均匀的流域",
                "min_stations": 1,
                "requires_watershed": False
            },
            {
                "name": "thiessen_polygon",
                "title": "泰森多边形法",
                "description": "基于Voronoi图的面积权重法，适用于站点分布不均的流域",
                "min_stations": 3,
                "requires_watershed": True
            },
            {
                "name": "inverse_distance",
                "title": "距离反比权重法",
                "description": "基于距离的权重插值法",
                "min_stations": 2,
                "requires_watershed": True
            },
            {
                "name": "kriging",
                "title": "克里金插值法",
                "description": "基于空间相关性的高精度插值法",
                "min_stations": 4,
                "requires_watershed": True
            }
        ]
    }
```

## 算法性能优化

### 并行计算实现

```python
# services/parallel_algorithms.py
import asyncio
import numpy as np
from concurrent.futures import ThreadPoolExecutor
from typing import List, Dict
import multiprocessing as mp

class ParallelRainfallService:
    """并行化的降雨算法服务"""
    
    def __init__(self, max_workers: int = None):
        self.max_workers = max_workers or mp.cpu_count()
        self.executor = ThreadPoolExecutor(max_workers=self.max_workers)
    
    async def parallel_thiessen_calculation(self, 
                                          stations_groups: List[List[RainfallStation]], 
                                          watersheds: List[Watershed]) -> List[Dict]:
        """并行计算多个流域的泰森多边形"""
        tasks = []
        
        for stations, watershed in zip(stations_groups, watersheds):
            task = asyncio.create_task(
                self._calculate_single_thiessen(stations, watershed)
            )
            tasks.append(task)
        
        results = await asyncio.gather(*tasks)
        return results
    
    async def _calculate_single_thiessen(self, 
                                       stations: List[RainfallStation], 
                                       watershed: Watershed) -> Dict:
        """单个流域的泰森多边形计算"""
        loop = asyncio.get_event_loop()
        
        # 在线程池中执行CPU密集型计算
        result = await loop.run_in_executor(
            self.executor,
            self._thiessen_computation,
            stations,
            watershed
        )
        
        return result
    
    def _thiessen_computation(self, 
                            stations: List[RainfallStation], 
                            watershed: Watershed) -> Dict:
        """泰森多边形的CPU密集型计算部分"""
        from scipy.spatial import Voronoi
        
        # 提取站点坐标
        points = np.array([[s.longitude, s.latitude] for s in stations])
        
        # 创建Voronoi图
        vor = Voronoi(points)
        
        # 计算权重和结果
        weights = self._compute_weights(vor, stations, watershed)
        weighted_sum = sum(
            station.rainfall * weight 
            for station, weight in zip(stations, weights)
        )
        
        return {
            "method": "thiessen_polygon_parallel",
            "result": round(weighted_sum, 2),
            "unit": "mm",
            "station_count": len(stations),
            "weights": [round(w, 4) for w in weights]
        }
    
    def _compute_weights(self, vor: Voronoi, 
                        stations: List[RainfallStation], 
                        watershed: Watershed) -> List[float]:
        """计算权重的优化实现"""
        # 使用NumPy向量化操作优化计算
        weights = np.zeros(len(stations))
        
        for i in range(len(stations)):
            region_index = vor.point_region[i]
            region = vor.regions[region_index]
            
            if -1 not in region and len(region) > 0:
                polygon_points = vor.vertices[region]
                area = self._fast_polygon_area(polygon_points)
                weights[i] = area
            else:
                weights[i] = watershed.area / len(stations)
        
        # 归一化
        total_weight = np.sum(weights)
        return weights / total_weight if total_weight > 0 else weights
    
    def _fast_polygon_area(self, points: np.ndarray) -> float:
        """优化的多边形面积计算"""
        if len(points) < 3:
            return 0
        
        # 使用NumPy的向量化操作
        x = points[:, 0]
        y = points[:, 1]
        
        # Shoelace公式的向量化实现
        area = 0.5 * np.abs(
            np.dot(x, np.roll(y, 1)) - np.dot(y, np.roll(x, 1))
        )
        
        return area
```

## 算法验证和测试

### 单元测试

```python
# tests/test_rainfall_algorithms.py
import pytest
import numpy as np
from services.rainfall_algorithms import RainfallAlgorithmService, RainfallStation, Watershed

class TestRainfallAlgorithms:
    
    @pytest.fixture
    def sample_stations(self):
        """测试用的降雨站点数据"""
        return [
            RainfallStation("S001", 120.0, 30.0, 25.5, quality_flag="A"),
            RainfallStation("S002", 120.1, 30.1, 30.2, quality_flag="A"),
            RainfallStation("S003", 120.2, 30.0, 22.8, quality_flag="B"),
            RainfallStation("S004", 120.1, 29.9, 28.1, quality_flag="A"),
        ]
    
    @pytest.fixture
    def sample_watershed(self):
        """测试用的流域数据"""
        return Watershed(
            id="W001",
            name="Test Watershed",
            area=100.0,
            boundary_points=[
                (119.9, 29.8), (120.3, 29.8), 
                (120.3, 30.2), (119.9, 30.2)
            ],
            center_point=(120.1, 30.0)
        )
    
    @pytest.mark.asyncio
    async def test_arithmetic_mean(self, sample_stations):
        """测试算术平均值法"""
        service = RainfallAlgorithmService()
        result = await service.arithmetic_mean(sample_stations)
        
        expected_mean = np.mean([25.5, 30.2, 22.8, 28.1])
        
        assert result["method"] == "arithmetic_mean"
        assert abs(result["result"] - expected_mean) < 0.01
        assert result["station_count"] == 4
        assert result["unit"] == "mm"
    
    @pytest.mark.asyncio
    async def test_thiessen_polygon(self, sample_stations, sample_watershed):
        """测试泰森多边形法"""
        service = RainfallAlgorithmService()
        result = await service.thiessen_polygon(sample_stations, sample_watershed)
        
        assert result["method"] == "thiessen_polygon"
        assert result["station_count"] == 4
        assert len(result["weights"]) == 4
        assert abs(sum(result["weights"]) - 1.0) < 0.001  # 权重和应为1
    
    @pytest.mark.asyncio
    async def test_inverse_distance_weighting(self, sample_stations, sample_watershed):
        """测试距离反比权重法"""
        service = RainfallAlgorithmService()
        result = await service.inverse_distance_weighting(
            sample_stations, sample_watershed, power=2.0
        )
        
        assert result["method"] == "inverse_distance_weighting"
        assert result["power"] == 2.0
        assert len(result["weights"]) == 4
        assert len(result["distances"]) == 4
        assert abs(sum(result["weights"]) - 1.0) < 0.001
    
    @pytest.mark.asyncio
    async def test_algorithm_with_insufficient_data(self, sample_watershed):
        """测试数据不足的情况"""
        service = RainfallAlgorithmService()
        
        # 只有一个站点，泰森多边形应该失败
        single_station = [RainfallStation("S001", 120.0, 30.0, 25.5)]
        
        with pytest.raises(Exception):
            await service.thiessen_polygon(single_station, sample_watershed)
    
    def test_distance_calculation(self):
        """测试距离计算函数"""
        service = RainfallAlgorithmService()
        
        # 测试已知距离
        distance = service._calculate_distance(0, 0, 0, 1)
        expected_distance = 111.32  # 大约111.32公里（1度纬度）
        
        assert abs(distance - expected_distance) < 1.0
    
    def test_polygon_area_calculation(self):
        """测试多边形面积计算"""
        service = RainfallAlgorithmService()
        
        # 测试单位正方形
        square_points = [(0, 0), (1, 0), (1, 1), (0, 1)]
        area = service._calculate_polygon_area(square_points)
        
        assert abs(area - 1.0) < 0.001
```

## 性能基准测试

### 基准测试实现

```python
# tests/benchmark_algorithms.py
import time
import asyncio
import numpy as np
from typing import List
from services.rainfall_algorithms import RainfallAlgorithmService, RainfallStation, Watershed

class AlgorithmBenchmark:
    """算法性能基准测试"""
    
    def __init__(self):
        self.service = RainfallAlgorithmService()
    
    def generate_test_data(self, num_stations: int) -> tuple:
        """生成测试数据"""
        np.random.seed(42)  # 确保结果可重现
        
        # 生成随机分布的站点
        stations = []
        for i in range(num_stations):
            lon = 120.0 + np.random.uniform(-0.5, 0.5)
            lat = 30.0 + np.random.uniform(-0.5, 0.5)
            rainfall = np.random.uniform(0, 50)
            
            stations.append(RainfallStation(
                id=f"S{i:03d}",
                longitude=lon,
                latitude=lat,
                rainfall=rainfall,
                quality_flag="A"
            ))
        
        # 生成测试流域
        watershed = Watershed(
            id="TEST_WATERSHED",
            name="Benchmark Watershed",
            area=100.0,
            boundary_points=[
                (119.5, 29.5), (120.5, 29.5),
                (120.5, 30.5), (119.5, 30.5)
            ],
            center_point=(120.0, 30.0)
        )
        
        return stations, watershed
    
    async def benchmark_algorithm(self, algorithm_name: str, 
                                stations: List[RainfallStation], 
                                watershed: Watershed, 
                                iterations: int = 100) -> Dict:
        """基准测试单个算法"""
        algorithm = getattr(self.service, algorithm_name)
        
        # 预热
        if algorithm_name == "arithmetic_mean":
            await algorithm(stations)
        else:
            await algorithm(stations, watershed)
        
        # 正式测试
        start_time = time.time()
        
        for _ in range(iterations):
            if algorithm_name == "arithmetic_mean":
                await algorithm(stations)
            else:
                await algorithm(stations, watershed)
        
        end_time = time.time()
        
        total_time = end_time - start_time
        avg_time = total_time / iterations
        
        return {
            "algorithm": algorithm_name,
            "total_time": round(total_time, 4),
            "average_time": round(avg_time, 4),
            "iterations": iterations,
            "stations_count": len(stations),
            "throughput": round(iterations / total_time, 2)  # 次/秒
        }
    
    async def run_comprehensive_benchmark(self):
        """运行综合基准测试"""
        algorithms = [
            "arithmetic_mean",
            "thiessen_polygon", 
            "inverse_distance_weighting",
            "kriging_interpolation"
        ]
        
        station_counts = [10, 50, 100, 200, 500]
        results = []
        
        for count in station_counts:
            print(f"Testing with {count} stations...")
            stations, watershed = self.generate_test_data(count)
            
            for algorithm in algorithms:
                try:
                    result = await self.benchmark_algorithm(
                        algorithm, stations, watershed, iterations=50
                    )
                    results.append(result)
                    print(f"  {algorithm}: {result['average_time']}s avg")
                except Exception as e:
                    print(f"  {algorithm}: Failed - {e}")
        
        return results
    
    def generate_performance_report(self, results: List[Dict]) -> str:
        """生成性能报告"""
        report = "# 算法性能基准测试报告\n\n"
        
        # 按站点数量分组
        by_stations = {}
        for result in results:
            count = result['stations_count']
            if count not in by_stations:
                by_stations[count] = []
            by_stations[count].append(result)
        
        for count in sorted(by_stations.keys()):
            report += f"## {count}个站点测试结果\n\n"
            report += "| 算法 | 平均耗时(s) | 吞吐量(次/s) |\n"
            report += "|------|-------------|-------------|\n"
            
            for result in by_stations[count]:
                report += f"| {result['algorithm']} | {result['average_time']} | {result['throughput']} |\n"
            
            report += "\n"
        
        return report

# 运行基准测试
async def main():
    benchmark = AlgorithmBenchmark()
    results = await benchmark.run_comprehensive_benchmark()
    
    report = benchmark.generate_performance_report(results)
    
    # 保存报告
    with open("algorithm_benchmark_report.md", "w", encoding="utf-8") as f:
        f.write(report)
    
    print("基准测试完成，报告已保存到 algorithm_benchmark_report.md")

if __name__ == "__main__":
    asyncio.run(main())
```

## 算法配置和管理

### 算法配置文件

```yaml
# config/algorithm_config.yaml
algorithms:
  arithmetic_mean:
    enabled: true
    default_params: {}
    validation:
      min_stations: 1
      max_stations: 1000
    
  thiessen_polygon:
    enabled: true
    default_params: {}
    validation:
      min_stations: 3
      max_stations: 500
    
  inverse_distance_weighting:
    enabled: true
    default_params:
      power: 2.0
    validation:
      min_stations: 2
      max_stations: 200
      power_range: [0.5, 5.0]
    
  kriging_interpolation:
    enabled: true
    default_params:
      method: "cubic"
      grid_size: 50
    validation:
      min_stations: 4
      max_stations: 100

performance:
  parallel_threshold: 50  # 超过50个站点时启用并行计算
  cache_results: true
  cache_ttl: 3600  # 缓存1小时

quality_control:
  valid_quality_flags: ["A", "B"]
  min_valid_ratio: 0.5  # 至少50%的站点数据质量合格
  outlier_detection: true
  outlier_threshold: 3.0  # 3倍标准差
```

### 算法管理器

```python
# services/algorithm_manager.py
import yaml
from typing import Dict, List, Optional
from services.rainfall_algorithms import RainfallAlgorithmService
from services.parallel_algorithms import ParallelRainfallService

class AlgorithmManager:
    """算法管理器"""
    
    def __init__(self, config_path: str = "config/algorithm_config.yaml"):
        with open(config_path, 'r', encoding='utf-8') as f:
            self.config = yaml.safe_load(f)
        
        self.service = RainfallAlgorithmService()
        self.parallel_service = ParallelRainfallService()
    
    def validate_request(self, algorithm_name: str, 
                        stations: List[RainfallStation],
                        **kwargs) -> bool:
        """验证算法请求"""
        if algorithm_name not in self.config['algorithms']:
            return False
        
        algo_config = self.config['algorithms'][algorithm_name]
        
        if not algo_config['enabled']:
            return False
        
        validation = algo_config['validation']
        
        # 检查站点数量
        station_count = len(stations)
        if (station_count < validation['min_stations'] or 
            station_count > validation['max_stations']):
            return False
        
        # 检查数据质量
        valid_flags = self.config['quality_control']['valid_quality_flags']
        valid_count = sum(1 for s in stations if s.quality_flag in valid_flags)
        valid_ratio = valid_count / station_count
        
        min_ratio = self.config['quality_control']['min_valid_ratio']
        if valid_ratio < min_ratio:
            return False
        
        return True
    
    async def execute_algorithm(self, algorithm_name: str,
                              stations: List[RainfallStation],
                              **kwargs) -> Dict:
        """执行算法"""
        if not self.validate_request(algorithm_name, stations, **kwargs):
            raise ValueError(f"Invalid request for algorithm {algorithm_name}")
        
        # 检查是否需要并行计算
        threshold = self.config['performance']['parallel_threshold']
        use_parallel = len(stations) > threshold
        
        # 应用异常值检测
        if self.config['quality_control']['outlier_detection']:
            stations = self._remove_outliers(stations)
        
        # 执行算法
        if use_parallel and hasattr(self.parallel_service, f"parallel_{algorithm_name}"):
            # 使用并行版本
            method = getattr(self.parallel_service, f"parallel_{algorithm_name}")
            result = await method(stations, **kwargs)
        else:
            # 使用标准版本
            method = getattr(self.service, algorithm_name)
            result = await method(stations, **kwargs)
        
        return result
    
    def _remove_outliers(self, stations: List[RainfallStation]) -> List[RainfallStation]:
        """移除异常值"""
        import numpy as np
        
        rainfall_values = [s.rainfall for s in stations]
        mean_val = np.mean(rainfall_values)
        std_val = np.std(rainfall_values)
        threshold = self.config['quality_control']['outlier_threshold']
        
        filtered_stations = []
        for station in stations:
            z_score = abs(station.rainfall - mean_val) / std_val
            if z_score <= threshold:
                filtered_stations.append(station)
        
        return filtered_stations
```