# DEM 数据自动下载工具

自动下载 SRTM 数字高程模型（DEM）数据，支持 Shapefile / GeoJSON 矢量范围裁切。

## 功能

- 支持 **SRTM1（30m 分辨率）** 和 **SRTM3（90m 分辨率）**
- 输入：Shapefile（`.shp`）或 GeoJSON（`.geojson` / `.json`）
- 双下载引擎：
  - **OpenTopography API**（需 API key，速度快，返回合并后的 GeoTIFF）
  - **HGT 瓦片直下**（无需 API key，自动回退，多线程下载后合并裁切）
- 输出：GeoTIFF 或原始 HGT 瓦片
- 配置文件驱动，支持命令行覆写入参

## 安装

```bash
pip install pyyaml geopandas rasterio requests tqdm
```

## 快速开始

### 1. 准备区域文件

创建一个 GeoJSON 文件，定义你要下载 DEM 的区域。例如 `area.geojson`：

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {},
      "geometry": {
        "type": "Polygon",
        "coordinates": [[[113.5, 22.3], [114.5, 22.3], [114.5, 23.2], [113.5, 23.2], [113.5, 22.3]]]
      }
    }
  ]
}
```

也支持 Shapefile（`.shp`）。

### 2. 修改配置文件

编辑 `config.yaml`：

```yaml
dem:
  source: srtm3              # srtm1（30m）或 srtm3（90m）

# 以下二选一：
# input:
#   shapefile: ./area.shp     # 使用 Shapefile
#   geojson: ./area.geojson   # 使用 GeoJSON

output:
  directory: ./dem_output     # 输出目录
  format: geotiff             # geotiff 或 hgt

downloader:
  max_workers: 4              # 并发下载数
  # opentopography_api_key: "your-key"   # 可选，有则走 API，无则自动回退
```

> 文件路径也可通过命令行 `-i` 传入，此时配置文件中 `input` 段可省略。

### 3. 运行

```bash
# 方式一：文件路径写在配置文件里
python -m dem_downloader.cli -c config.yaml

# 方式二：通过命令行传入文件路径（推荐）
python -m dem_downloader.cli -c config.yaml -i ./area.geojson
```

### 运行输出示例

```
Loaded config: Config(dem=srtm3, input=./area.geojson, output=./dem_output/geotiff, max_workers=4)
Bounding box: (113.5, 22.3, 114.5, 23.2)
Resolved to 4 SRTM tile(s)
Downloading tiles: 100%|████████| 4/4 [00:03<00:00]
Merging tiles...
Merged DEM saved to: dem_output\dem_113.50_22.30_114.50_23.20.tif

Done! Output: dem_output\dem_113.50_22.30_114.50_23.20.tif
```

## 配置文件详解

| 配置项 | 说明 | 默认值 |
|---|---|---|
| `dem.source` | 数据源：`srtm1`（30m）或 `srtm3`（90m） | `srtm3` |
| `input.shapefile` | Shapefile 路径（与 `geojson` 二选一） | 无 |
| `input.geojson` | GeoJSON 路径（与 `shapefile` 二选一） | 无 |
| `output.directory` | 输出目录 | `./dem_output` |
| `output.format` | 输出格式：`geotiff` 或 `hgt` | `geotiff` |
| `downloader.max_workers` | HGT 瓦片并发下载数 | `4` |
| `downloader.opentopography_api_key` | OpenTopography API 密钥（可选） | 无 |

## 命令行参数

```
python -m dem_downloader.cli -c <配置文件> [-i <输入文件>] [--verbose]
```

| 参数 | 说明 |
|---|---|
| `-c, --config` | **必填**，配置文件路径 |
| `-i, --input` | 可选，输入文件路径。覆写配置文件中的 `input` 段 |
| `--verbose` | 可选，显示详细日志 |

## 获取 OpenTopography API Key（可选）

1. 打开 https://opentopography.org/
2. 注册账号
3. 进入 Dashboard → My API Keys → 生成新 key
4. 填到 `config.yaml` 的 `opentopography_api_key` 字段

**没有 key 也能用。** 程序会自动检测：有 key 走 OpenTopography API（一次请求直接拿到合并好的 GeoTIFF），没 key 自动回退到 HGT 瓦片直下模式（从公开镜像下载逐块合并）。

## 输出说明

- **GeoTIFF 格式**：一个完整的、已按输入范围裁切的 DEM 文件，可直接在 QGIS、ArcGIS、Global Mapper 等 GIS 软件中打开
- **HGT 格式**：原始 SRTM 瓦片文件（每格 1°×1°），适合需要原始数据的场景

## 常见问题

**Q: 下载速度慢怎么办？**
- 配置 API key（走 OpenTopography API 更快）
- 调大 `max_workers`（增加并发数）
- 网络原因可换时间重试

**Q: 部分瓦片下载失败？**
程序会自动重试其他镜像地址，大多数情况下能成功。如果仍有失败，检查网络连接。

**Q: 下载的 DEM 范围不对？**
检查输入 Shapefile/GeoJSON 的坐标系，确保是 WGS84（EPSG:4326），单位是经纬度。

**Q: 能不能批量下载？**
可以。写多个配置文件，或用脚本循环调用即可。
