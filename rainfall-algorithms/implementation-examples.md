# 面平均雨量算法实现示例

## 核心算法实现

### 1. 算术平均值法实现

#### Python实现
```python
import numpy as np
from typing import List, Dict, Optional
from dataclasses import dataclass
import logging

@dataclass
class RainfallStation:
    """雨量站数据结构"""
    id: str
    name: str
    longitude: float
    latitude: float
    rainfall: float
    timestamp: str
    elevation: Optional[float] = None

@dataclass
class ArithmeticMeanResult:
    """算术平均值计算结果"""
    average_rainfall: float
    station_count: int
    method: str = "arithmetic_mean"
    unit: str = "mm"
    calculation_time: str = None
    stations_used: List[str] = None

class ArithmeticMeanCalculator:
    """算术平均值计算器"""
    
    def __init__(self):
        self.logger = logging.getLogger(__name__)
    
    def validate_data(self, stations: List[RainfallStation]) -> bool:
        """数据验证"""
        if not stations:
            raise ValueError("雨量站数据不能为空")
        
        for station in stations:
            if station.rainfall < 0 or station.rainfall > 1000:
                raise ValueError(f"站点{station.id}降雨量异常: {station.rainfall}mm")
            
            if not (-180 <= station.longitude <= 180):
                raise ValueError(f"站点{station.id}经度无效: {station.longitude}")
            
            if not (-90 <= station.latitude <= 90):
                raise ValueError(f"站点{station.id}纬度无效: {station.latitude}")
        
        return True
    
    def calculate(self, stations: List[RainfallStation]) -> ArithmeticMeanResult:
        """计算算术平均雨量"""
        self.validate_data(stations)
        
        # 提取降雨量数据
        rainfall_values = [station.rainfall for station in stations]
        
        # 计算平均值
        average_rainfall = np.mean(rainfall_values)
        
        # 构建结果
        result = ArithmeticMeanResult(
            average_rainfall=round(average_rainfall, 2),
            station_count=len(stations),
            stations_used=[station.id for station in stations]
        )
        
        self.logger.info(f"算术平均值计算完成: {result.average_rainfall}mm")
        return result

# 使用示例
def example_arithmetic_mean():
    """算术平均值使用示例"""
    stations = [
        RainfallStation("S001", "站点1", 116.3974, 39.9093, 12.5, "2024-01-01T08:00:00Z"),
        RainfallStation("S002", "站点2", 116.4074, 39.9193, 15.2, "2024-01-01T08:00:00Z"),
        RainfallStation("S003", "站点3", 116.3874, 39.8993, 11.8, "2024-01-01T08:00:00Z"),
        RainfallStation("S004", "站点4", 116.4174, 39.9293, 14.6, "2024-01-01T08:00:00Z"),
        RainfallStation("S005", "站点5", 116.3774, 39.8893, 13.1, "2024-01-01T08:00:00Z")
    ]
    
    calculator = ArithmeticMeanCalculator()
    result = calculator.calculate(stations)
    
    print(f"流域面平均雨量: {result.average_rainfall} mm")
    print(f"参与计算站点数: {result.station_count}")
    return result
```

### 2. 泰森多边形法实现

#### Python实现
```python
import numpy as np
from scipy.spatial import Voronoi
from shapely.geometry import Polygon, Point
from shapely.ops import unary_union
import geopandas as gpd
from typing import List, Tuple
from dataclasses import dataclass

@dataclass
class ThiessenPolygonResult:
    """泰森多边形计算结果"""
    average_rainfall: float
    station_count: int
    watershed_area: float
    method: str = "thiessen_polygon"
    unit: str = "mm"
    weighted_contributions: List[Dict] = None

class ThiessenPolygonCalculator:
    """泰森多边形计算器"""
    
    def __init__(self):
        self.logger = logging.getLogger(__name__)
    
    def create_voronoi_polygons(self, stations: List[RainfallStation]) -> List[Polygon]:
        """创建Voronoi多边形"""
        # 提取坐标点
        points = np.array([[s.longitude, s.latitude] for s in stations])
        
        # 创建Voronoi图
        vor = Voronoi(points)
        
        polygons = []
        for i, point_region in enumerate(vor.point_region):
            region = vor.regions[point_region]
            if -1 not in region and len(region) > 0:
                # 构建多边形
                polygon_coords = [vor.vertices[j] for j in region]
                if len(polygon_coords) >= 3:
                    polygon = Polygon(polygon_coords)
                    polygons.append(polygon)
                else:
                    # 处理无界区域
                    polygons.append(self._create_bounded_polygon(vor, i, points))
            else:
                # 处理无界区域
                polygons.append(self._create_bounded_polygon(vor, i, points))
        
        return polygons
    
    def _create_bounded_polygon(self, vor, point_idx: int, points: np.ndarray) -> Polygon:
        """为无界区域创建有界多边形"""
        # 简化实现：创建一个大的边界框
        min_x, min_y = points.min(axis=0) - 1
        max_x, max_y = points.max(axis=0) + 1
        
        # 返回边界框多边形
        return Polygon([(min_x, min_y), (max_x, min_y), (max_x, max_y), (min_x, max_y)])
    
    def calculate_intersection_areas(self, polygons: List[Polygon], 
                                   watershed_boundary: Polygon) -> List[float]:
        """计算多边形与流域边界的交集面积"""
        areas = []
        for polygon in polygons:
            try:
                intersection = polygon.intersection(watershed_boundary)
                area = intersection.area if intersection.is_valid else 0.0
                areas.append(area)
            except Exception as e:
                self.logger.warning(f"面积计算错误: {e}")
                areas.append(0.0)
        
        return areas
    
    def calculate(self, stations: List[RainfallStation], 
                 watershed_boundary: Polygon) -> ThiessenPolygonResult:
        """计算泰森多边形加权平均雨量"""
        if len(stations) < 3:
            raise ValueError("泰森多边形法至少需要3个雨量站")
        
        # 创建Voronoi多边形
        polygons = self.create_voronoi_polygons(stations)
        
        # 计算交集面积
        areas = self.calculate_intersection_areas(polygons, watershed_boundary)
        
        # 计算加权平均
        total_area = sum(areas)
        if total_area == 0:
            raise ValueError("有效面积为零，请检查流域边界和站点位置")
        
        weighted_sum = sum(station.rainfall * area 
                          for station, area in zip(stations, areas))
        average_rainfall = weighted_sum / total_area
        
        # 构建详细结果
        contributions = []
        for i, (station, area) in enumerate(zip(stations, areas)):
            contribution = {
                "station_id": station.id,
                "rainfall": station.rainfall,
                "area": round(area, 3),
                "area_weight": round(area / total_area, 4),
                "contribution": round(station.rainfall * area / total_area, 3)
            }
            contributions.append(contribution)
        
        result = ThiessenPolygonResult(
            average_rainfall=round(average_rainfall, 2),
            station_count=len(stations),
            watershed_area=round(total_area, 2),
            weighted_contributions=contributions
        )
        
        self.logger.info(f"泰森多边形计算完成: {result.average_rainfall}mm")
        return result

# 使用示例
def example_thiessen_polygon():
    """泰森多边形使用示例"""
    stations = [
        RainfallStation("S001", "站点1", 116.3974, 39.9093, 12.5, "2024-01-01T08:00:00Z"),
        RainfallStation("S002", "站点2", 116.4074, 39.9193, 15.2, "2024-01-01T08:00:00Z"),
        RainfallStation("S003", "站点3", 116.3874, 39.8993, 11.8, "2024-01-01T08:00:00Z"),
        RainfallStation("S004", "站点4", 116.4174, 39.9293, 14.6, "2024-01-01T08:00:00Z")
    ]
    
    # 定义流域边界（示例）
    watershed_coords = [
        (116.35, 39.85), (116.45, 39.85), 
        (116.45, 39.95), (116.35, 39.95), (116.35, 39.85)
    ]
    watershed_boundary = Polygon(watershed_coords)
    
    calculator = ThiessenPolygonCalculator()
    result = calculator.calculate(stations, watershed_boundary)
    
    print(f"流域面平均雨量: {result.average_rainfall} mm")
    print(f"流域总面积: {result.watershed_area} km²")
    
    return result
```

## API接口设计

### 1. RESTful API接口

#### 算术平均值接口
```http
POST /api/v1/rainfall/arithmetic-mean
Content-Type: application/json

{
  "stations": [
    {
      "id": "S001",
      "name": "站点1",
      "longitude": 116.3974,
      "latitude": 39.9093,
      "rainfall": 12.5,
      "timestamp": "2024-01-01T08:00:00Z",
      "elevation": 50.0
    }
  ],
  "options": {
    "validate_data": true,
    "include_details": true
  }
}
```

**响应格式：**
```json
{
  "success": true,
  "data": {
    "method": "arithmetic_mean",
    "average_rainfall": 13.44,
    "unit": "mm",
    "station_count": 5,
    "calculation_time": "2024-01-01T08:05:00Z",
    "stations_used": ["S001", "S002", "S003", "S004", "S005"],
    "statistics": {
      "min_rainfall": 11.8,
      "max_rainfall": 15.2,
      "std_deviation": 1.34
    }
  },
  "metadata": {
    "processing_time_ms": 15,
    "algorithm_version": "1.0.0",
    "api_version": "v1"
  }
}
```

#### 泰森多边形接口
```http
POST /api/v1/rainfall/thiessen-polygon
Content-Type: application/json

{
  "stations": [
    {
      "id": "S001",
      "name": "站点1",
      "longitude": 116.3974,
      "latitude": 39.9093,
      "rainfall": 12.5,
      "timestamp": "2024-01-01T08:00:00Z"
    }
  ],
  "watershed": {
    "boundary": "POLYGON((116.35 39.85, 116.45 39.85, 116.45 39.95, 116.35 39.95, 116.35 39.85))",
    "area": 1000.5,
    "name": "测试流域"
  },
  "options": {
    "include_polygons": false,
    "precision": 2
  }
}
```

**响应格式：**
```json
{
  "success": true,
  "data": {
    "method": "thiessen_polygon",
    "average_rainfall": 13.52,
    "unit": "mm",
    "station_count": 4,
    "watershed_area": 127.3,
    "calculation_time": "2024-01-01T08:05:00Z",
    "weighted_contributions": [
      {
        "station_id": "S001",
        "rainfall": 12.5,
        "area": 25.6,
        "area_weight": 0.201,
        "contribution": 2.51
      }
    ]
  },
  "metadata": {
    "processing_time_ms": 245,
    "algorithm_version": "1.0.0",
    "api_version": "v1"
  }
}
```

### 2. Flask API实现

```python
from flask import Flask, request, jsonify
from flask_cors import CORS
import logging
from datetime import datetime
import time

app = Flask(__name__)
CORS(app)

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.route('/api/v1/rainfall/arithmetic-mean', methods=['POST'])
def calculate_arithmetic_mean():
    """算术平均值计算接口"""
    try:
        start_time = time.time()
        data = request.get_json()
        
        # 验证输入数据
        if 'stations' not in data or not data['stations']:
            return jsonify({
                'success': False,
                'error': {
                    'code': 'INVALID_INPUT',
                    'message': '缺少雨量站数据'
                }
            }), 400
        
        # 解析雨量站数据
        stations = []
        for station_data in data['stations']:
            station = RainfallStation(
                id=station_data['id'],
                name=station_data['name'],
                longitude=station_data['longitude'],
                latitude=station_data['latitude'],
                rainfall=station_data['rainfall'],
                timestamp=station_data['timestamp'],
                elevation=station_data.get('elevation')
            )
            stations.append(station)
        
        # 执行计算
        calculator = ArithmeticMeanCalculator()
        result = calculator.calculate(stations)
        
        # 计算统计信息
        rainfall_values = [s.rainfall for s in stations]
        statistics = {
            'min_rainfall': min(rainfall_values),
            'max_rainfall': max(rainfall_values),
            'std_deviation': round(np.std(rainfall_values), 2)
        }
        
        processing_time = round((time.time() - start_time) * 1000, 2)
        
        return jsonify({
            'success': True,
            'data': {
                'method': result.method,
                'average_rainfall': result.average_rainfall,
                'unit': result.unit,
                'station_count': result.station_count,
                'calculation_time': datetime.utcnow().isoformat() + 'Z',
                'stations_used': result.stations_used,
                'statistics': statistics
            },
            'metadata': {
                'processing_time_ms': processing_time,
                'algorithm_version': '1.0.0',
                'api_version': 'v1'
            }
        })
        
    except ValueError as e:
        return jsonify({
            'success': False,
            'error': {
                'code': 'VALIDATION_ERROR',
                'message': str(e)
            }
        }), 400
    
    except Exception as e:
        logger.error(f"计算错误: {e}")
        return jsonify({
            'success': False,
            'error': {
                'code': 'CALCULATION_ERROR',
                'message': '计算过程中发生错误'
            }
        }), 500

@app.route('/api/v1/rainfall/thiessen-polygon', methods=['POST'])
def calculate_thiessen_polygon():
    """泰森多边形计算接口"""
    try:
        start_time = time.time()
        data = request.get_json()
        
        # 验证输入数据
        if 'stations' not in data or not data['stations']:
            return jsonify({
                'success': False,
                'error': {
                    'code': 'INVALID_INPUT',
                    'message': '缺少雨量站数据'
                }
            }), 400
        
        if 'watershed' not in data or 'boundary' not in data['watershed']:
            return jsonify({
                'success': False,
                'error': {
                    'code': 'INVALID_INPUT',
                    'message': '缺少流域边界数据'
                }
            }), 400
        
        # 解析数据
        stations = []
        for station_data in data['stations']:
            station = RainfallStation(
                id=station_data['id'],
                name=station_data['name'],
                longitude=station_data['longitude'],
                latitude=station_data['latitude'],
                rainfall=station_data['rainfall'],
                timestamp=station_data['timestamp']
            )
            stations.append(station)
        
        # 解析流域边界（简化实现）
        from shapely import wkt
        watershed_boundary = wkt.loads(data['watershed']['boundary'])
        
        # 执行计算
        calculator = ThiessenPolygonCalculator()
        result = calculator.calculate(stations, watershed_boundary)
        
        processing_time = round((time.time() - start_time) * 1000, 2)
        
        return jsonify({
            'success': True,
            'data': {
                'method': result.method,
                'average_rainfall': result.average_rainfall,
                'unit': result.unit,
                'station_count': result.station_count,
                'watershed_area': result.watershed_area,
                'calculation_time': datetime.utcnow().isoformat() + 'Z',
                'weighted_contributions': result.weighted_contributions
            },
            'metadata': {
                'processing_time_ms': processing_time,
                'algorithm_version': '1.0.0',
                'api_version': 'v1'
            }
        })
        
    except Exception as e:
        logger.error(f"计算错误: {e}")
        return jsonify({
            'success': False,
            'error': {
                'code': 'CALCULATION_ERROR',
                'message': str(e)
            }
        }), 500

@app.route('/api/v1/health', methods=['GET'])
def health_check():
    """健康检查接口"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.utcnow().isoformat() + 'Z',
        'version': '1.0.0'
    })

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=8080)
```

## 批处理接口

### 批量计算实现
```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import List, Dict, Any
import asyncio

class BatchRainfallCalculator:
    """批量降雨计算器"""
    
    def __init__(self, max_workers: int = 4):
        self.max_workers = max_workers
        self.arithmetic_calculator = ArithmeticMeanCalculator()
        self.thiessen_calculator = ThiessenPolygonCalculator()
    
    def batch_calculate(self, requests: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
        """批量计算"""
        results = []
        
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            # 提交所有任务
            future_to_request = {}
            for i, request in enumerate(requests):
                future = executor.submit(self._process_single_request, request)
                future_to_request[future] = i
            
            # 收集结果
            for future in as_completed(future_to_request):
                request_index = future_to_request[future]
                try:
                    result = future.result()
                    result['request_index'] = request_index
                    results.append(result)
                except Exception as e:
                    error_result = {
                        'request_index': request_index,
                        'success': False,
                        'error': str(e)
                    }
                    results.append(error_result)
        
        # 按原始顺序排序
        results.sort(key=lambda x: x['request_index'])
        return results
    
    def _process_single_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """处理单个请求"""
        method = request.get('method', 'arithmetic_mean')
        stations_data = request['stations']
        
        # 构建站点对象
        stations = []
        for station_data in stations_data:
            station = RainfallStation(**station_data)
            stations.append(station)
        
        if method == 'arithmetic_mean':
            result = self.arithmetic_calculator.calculate(stations)
            return {
                'success': True,
                'method': result.method,
                'average_rainfall': result.average_rainfall,
                'station_count': result.station_count
            }
        
        elif method == 'thiessen_polygon':
            from shapely import wkt
            watershed_boundary = wkt.loads(request['watershed']['boundary'])
            result = self.thiessen_calculator.calculate(stations, watershed_boundary)
            return {
                'success': True,
                'method': result.method,
                'average_rainfall': result.average_rainfall,
                'station_count': result.station_count,
                'watershed_area': result.watershed_area
            }
        
        else:
            raise ValueError(f"不支持的计算方法: {method}")

# 批处理API接口
@app.route('/api/v1/rainfall/batch', methods=['POST'])
def batch_calculate():
    """批量计算接口"""
    try:
        data = request.get_json()
        requests = data.get('requests', [])
        
        if not requests:
            return jsonify({
                'success': False,
                'error': {
                    'code': 'INVALID_INPUT',
                    'message': '缺少计算请求'
                }
            }), 400
        
        # 执行批量计算
        calculator = BatchRainfallCalculator()
        results = calculator.batch_calculate(requests)
        
        return jsonify({
            'success': True,
            'data': {
                'total_requests': len(requests),
                'results': results
            }
        })
        
    except Exception as e:
        logger.error(f"批量计算错误: {e}")
        return jsonify({
            'success': False,
            'error': {
                'code': 'BATCH_CALCULATION_ERROR',
                'message': str(e)
            }
        }), 500
```

## 性能优化

### 1. 缓存机制
```python
from functools import lru_cache
import hashlib
import json

class CachedRainfallCalculator:
    """带缓存的降雨计算器"""
    
    def __init__(self, cache_size: int = 1000):
        self.cache_size = cache_size
        self.arithmetic_calculator = ArithmeticMeanCalculator()
    
    def _generate_cache_key(self, stations: List[RainfallStation]) -> str:
        """生成缓存键"""
        station_data = []
        for station in sorted(stations, key=lambda x: x.id):
            station_data.append({
                'id': station.id,
                'rainfall': station.rainfall,
                'longitude': station.longitude,
                'latitude': station.latitude
            })
        
        data_str = json.dumps(station_data, sort_keys=True)
        return hashlib.md5(data_str.encode()).hexdigest()
    
    @lru_cache(maxsize=1000)
    def _cached_arithmetic_mean(self, cache_key: str, rainfall_tuple: tuple) -> float:
        """缓存的算术平均值计算"""
        return sum(rainfall_tuple) / len(rainfall_tuple)
    
    def calculate_arithmetic_mean(self, stations: List[RainfallStation]) -> ArithmeticMeanResult:
        """带缓存的算术平均值计算"""
        cache_key = self._generate_cache_key(stations)
        rainfall_tuple = tuple(station.rainfall for station in stations)
        
        average_rainfall = self._cached_arithmetic_mean(cache_key, rainfall_tuple)
        
        return ArithmeticMeanResult(
            average_rainfall=round(average_rainfall, 2),
            station_count=len(stations),
            stations_used=[station.id for station in stations]
        )
```

### 2. 异步处理
```python
import asyncio
from typing import Awaitable

class AsyncRainfallCalculator:
    """异步降雨计算器"""
    
    async def calculate_arithmetic_mean_async(self, 
                                            stations: List[RainfallStation]) -> ArithmeticMeanResult:
        """异步算术平均值计算"""
        # 模拟异步计算
        await asyncio.sleep(0.001)  # 模拟I/O操作
        
        rainfall_values = [station.rainfall for station in stations]
        average_rainfall = sum(rainfall_values) / len(rainfall_values)
        
        return ArithmeticMeanResult(
            average_rainfall=round(average_rainfall, 2),
            station_count=len(stations),
            stations_used=[station.id for station in stations]
        )
    
    async def batch_calculate_async(self, 
                                  requests: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
        """异步批量计算"""
        tasks = []
        for request in requests:
            task = self._process_request_async(request)
            tasks.append(task)
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        processed_results = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                processed_results.append({
                    'request_index': i,
                    'success': False,
                    'error': str(result)
                })
            else:
                result['request_index'] = i
                processed_results.append(result)
        
        return processed_results
    
    async def _process_request_async(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """异步处理单个请求"""
        stations_data = request['stations']
        stations = [RainfallStation(**data) for data in stations_data]
        
        result = await self.calculate_arithmetic_mean_async(stations)
        
        return {
            'success': True,
            'method': result.method,
            'average_rainfall': result.average_rainfall,
            'station_count': result.station_count
        }
```

## 测试用例

### 单元测试
```python
import unittest
import numpy as np

class TestRainfallCalculators(unittest.TestCase):
    """降雨计算器测试"""
    
    def setUp(self):
        """测试数据准备"""
        self.test_stations = [
            RainfallStation("S001", "站点1", 116.3974, 39.9093, 12.5, "2024-01-01T08:00:00Z"),
            RainfallStation("S002", "站点2", 116.4074, 39.9193, 15.2, "2024-01-01T08:00:00Z"),
            RainfallStation("S003", "站点3", 116.3874, 39.8993, 11.8, "2024-01-01T08:00:00Z"),
            RainfallStation("S004", "站点4", 116.4174, 39.9293, 14.6, "2024-01-01T08:00:00Z"),
            RainfallStation("S005", "站点5", 116.3774, 39.8893, 13.1, "2024-01-01T08:00:00Z")
        ]
    
    def test_arithmetic_mean_calculation(self):
        """测试算术平均值计算"""
        calculator = ArithmeticMeanCalculator()
        result = calculator.calculate(self.test_stations)
        
        expected_average = (12.5 + 15.2 + 11.8 + 14.6 + 13.1) / 5
        
        self.assertEqual(result.average_rainfall, round(expected_average, 2))
        self.assertEqual(result.station_count, 5)
        self.assertEqual(result.method, "arithmetic_mean")
    
    def test_data_validation(self):
        """测试数据验证"""
        calculator = ArithmeticMeanCalculator()
        
        # 测试空数据
        with self.assertRaises(ValueError):
            calculator.calculate([])
        
        # 测试异常降雨量
        invalid_station = RainfallStation("S001", "站点1", 116.3974, 39.9093, -5.0, "2024-01-01T08:00:00Z")
        with self.assertRaises(ValueError):
            calculator.calculate([invalid_station])
        
        # 测试异常坐标
        invalid_coord_station = RainfallStation("S001", "站点1", 200.0, 39.9093, 12.5, "2024-01-01T08:00:00Z")
        with self.assertRaises(ValueError):
            calculator.calculate([invalid_coord_station])
    
    def test_thiessen_polygon_basic(self):
        """测试泰森多边形基本功能"""
        calculator = ThiessenPolygonCalculator()
        
        # 创建简单的流域边界
        watershed_coords = [
            (116.35, 39.85), (116.45, 39.85), 
            (116.45, 39.95), (116.35, 39.95), (116.35, 39.85)
        ]
        watershed_boundary = Polygon(watershed_coords)
        
        # 使用前4个站点进行测试
        test_stations = self.test_stations[:4]
        
        result = calculator.calculate(test_stations, watershed_boundary)
        
        self.assertIsInstance(result.average_rainfall, float)
        self.assertEqual(result.station_count, 4)
        self.assertEqual(result.method, "thiessen_polygon")
        self.assertGreater(result.watershed_area, 0)

if __name__ == '__main__':
    unittest.main()
```

### 性能测试
```python
import time
import random

def performance_test():
    """性能测试"""
    # 生成测试数据
    def generate_test_stations(count: int) -> List[RainfallStation]:
        stations = []
        for i in range(count):
            station = RainfallStation(
                id=f"S{i:03d}",
                name=f"站点{i+1}",
                longitude=116.0 + random.uniform(-0.5, 0.5),
                latitude=39.0 + random.uniform(-0.5, 0.5),
                rainfall=random.uniform(0, 50),
                timestamp="2024-01-01T08:00:00Z"
            )
            stations.append(station)
        return stations
    
    # 测试不同规模的数据
    test_sizes = [10, 50, 100, 500, 1000]
    
    print("性能测试结果:")
    print("站点数量\t算术平均法(ms)\t泰森多边形法(ms)\t效率比")
    print("-" * 60)
    
    for size in test_sizes:
        stations = generate_test_stations(size)
        
        # 算术平均法性能测试
        calculator1 = ArithmeticMeanCalculator()
        start_time = time.time()
        for _ in range(100):  # 重复100次
            result1 = calculator1.calculate(stations)
        arithmetic_time = (time.time() - start_time) * 10  # 平均时间(ms)
        
        # 泰森多边形法性能测试（仅测试前50个站点以避免过长时间）
        if size <= 50:
            calculator2 = ThiessenPolygonCalculator()
            watershed_boundary = Polygon([
                (115.5, 38.5), (116.5, 38.5), 
                (116.5, 39.5), (115.5, 39.5), (115.5, 38.5)
            ])
            
            start_time = time.time()
            for _ in range(10):  # 重复10次
                result2 = calculator2.calculate(stations[:min(size, 20)], watershed_boundary)
            thiessen_time = (time.time() - start_time) * 100  # 平均时间(ms)
            
            efficiency_ratio = thiessen_time / arithmetic_time if arithmetic_time > 0 else 0
            
            print(f"{size}\t\t{arithmetic_time:.2f}\t\t{thiessen_time:.2f}\t\t{efficiency_ratio:.1f}")
        else:
            print(f"{size}\t\t{arithmetic_time:.2f}\t\t-\t\t-")

if __name__ == '__main__':
    performance_test()
```

## 部署配置

### Docker配置
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libgeos-dev \
    libproj-dev \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装Python依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 暴露端口
EXPOSE 8080

# 启动命令
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

### requirements.txt
```txt
Flask==2.3.3
Flask-CORS==4.0.0
gunicorn==21.2.0
numpy==1.24.3
scipy==1.11.1
shapely==2.0.1
geopandas==0.13.2
pandas==2.0.3
requests==2.31.0
python-dateutil==2.8.2
structlog==23.1.0
prometheus-client==0.17.1
```

这个实现示例提供了完整的面平均雨量算法实现，包括核心算法、API接口、性能优化、测试用例和部署配置，为知汛项目的降雨分析功能提供了坚实的技术基础。