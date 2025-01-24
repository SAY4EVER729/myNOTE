---
date: 2024-12-27
---
```
from datetime import datetime, timedelta

import json

import os

import re

from types import TracebackType

from typing import Any, AsyncGenerator, Type, Optional, Tuple

from anyforce import logging

from anyforce.logging import context

from pydantic_settings import SettingsConfigDict

import asyncio

  

from ..service import redis_nas as redis

from .typing import BaseSetting, File, Processor

from ..tortoise.dsp.dynamic_material_template import DynamicMaterialTemplate

from ..tortoise.dsp.behavior_log import BehaviorLog

from ..tortoise.xapi.user import User

from .deduplicator import Deduplicator

from .dynamic_material.dimension_info import DimensionInfo

from .dynamic_material.layer import get_dynamic_layers, SimplePSD

from .dynamic_material.render import Render

from ..tortoise import coro

  
  

logger = logging.getLogger(__name__)

  
  

class DynamicMaterialSetting(BaseSetting):

model_config = SettingsConfigDict(env_prefix="nas_dynamic_")

home: str = "创意中心/物料库/动态素材/"

overwrite: bool = False

timeout: int = 600

dry_run: bool = False

dry_run_prefix: str = ""

  
  

class DynamicMaterial(Processor):

def __init__(self, setting: DynamicMaterialSetting) -> None:

self.pattern = setting.compiled_pattern

self.overwrite = setting.overwrite

self.home = setting.home

self.timeout = setting.timeout

self.deduplicator = None

self.redis = None

self.dry_run = setting.dry_run

self.dry_run_prefix = setting.dry_run_prefix

  

async def __aenter__(self) -> Processor:

# 根据 dry_run 模式使用不同的 Redis 配置

if self.dry_run:

self.redis = redis.from_url("redis://localhost:6379/0")

else:

self.redis = redis

# 初始化去重器

self.deduplicator = Deduplicator(

redis=self.redis,

queue_name="dynamic_material",

consume_group_name="processor",

ex=86400, # 1天内不重复处理 NOTE: 暂未考虑超过1天且已经处理过的文件

)

return self

  

async def __aexit__(

self,

exc_type: Type[BaseException] | None,

exc_value: BaseException | None,

traceback: TracebackType | None,

) -> None:

return None

  

def _get_real_path(self, path: str) -> str:

"""根据是否是dry_run模式返回实际路径"""

if self.dry_run and not path.startswith(self.dry_run_prefix):

return "".join([self.dry_run_prefix, path])

return path

  

def _remove_dry_run_prefix(self, path: str) -> str:

if self.dry_run and path.startswith(self.dry_run_prefix):

return path[len(self.dry_run_prefix) :].lstrip("/")

return path

  

async def _get_template_for_path(

self, file_path: str

) -> AsyncGenerator[DynamicMaterialTemplate, None]:

"""根据文件路径获取匹配的模板"""

templates = await DynamicMaterialTemplate.filter(

layout__not_isnull=True,

)

logger.info(f"Found : {file_path}")

matched_template = None

  

# 检查文件是否在模板的input_path目录下

for template in templates:

# 如果path起始路径为 /创意中心/物料库/动态素材/user_name/自定义名字

path = template.input_path

  

if not path:

continue

  

real_path = self._get_real_path(path)

file_path = self._get_real_path(file_path)

logger.with_field(real_path=real_path, file_path=file_path).info("paths info")

if file_path.startswith(real_path):

matched_template = template

logger.with_field(matched_template=matched_template).info(

"matched_template"

)

yield matched_template

  

async def _ensure_output_path(self, template: DynamicMaterialTemplate) -> None:

# 如果目录不在 /创意中心/物料库/动态素材/ 下，将 output_path 改成统一默认路径 /创意中心/物料库/动态素材/tmp/user_name/

if not template.output_path.startswith("/创意中心/物料库/动态素材/"):

operater_id = await BehaviorLog.filter(

entity_id=template.id,

created_at__gte=datetime.now() - timedelta(hours=1),

model="DynamicMaterialTemplate",

).first()

if not operater_id:

# 如果找不到操作人，则放在 张晨/tmp/ 下

operater_id = 47

user = await User.filter(id=operater_id).first()

assert user

template.output_path = f"/创意中心/物料库/动态素材/{user.name}/tmp"

real_output_path = self._get_real_path(template.output_path)

if not os.path.exists(real_output_path):

os.makedirs(real_output_path, exist_ok=True)

logger.success(f"ensure_output_path: {template.output_path}")

  

def _extract_dimensions_from_filename(

self, filename: str

) -> Optional[Tuple[int, int]]:

"""从文件名中提取宽高信息"""

patterns = [

r"(\d+)[\*x_](\d+)", # 匹配300*300, 300x300, 300_300

r"(\d+)(?:width|w|宽|宽度).*?(\d+)(?:height|h|高|高度)", # 匹配300width400height

r"(?:width|w|宽|宽度)(\d+).*?(?:height|h|高|高度)(\d+)", # 匹配width300height400

]

# 先尝试使用patterns匹配

for pattern in patterns:

if match := re.search(pattern, filename, re.IGNORECASE):

try:

width = int(match.group(1))

height = int(match.group(2))

# 基本的有效性检查

if width >= 30 and height >= 30:

return width, height

except ValueError:

continue

  

# 如果patterns匹配失败，尝试查找所有数字对

numbers = re.findall(r"(\d{2,4})[\*x_](\d{2,4})", filename)

for width_str, height_str in numbers:

try:

width = int(width_str)

height = int(height_str)

# 基本的有效性检查

if width >= 30 and height >= 30:

return width, height

except ValueError:

continue

  

return None

  

def _is_size_in_range(

self,

target_width: float | int,

target_height: float | int,

file_width: float | int,

file_height: float | int,

ratio_deviation: float | int,

) -> bool:

if ratio_deviation < 1:

ratio_deviation = 1 / ratio_deviation

# ratio_deviation为1时表示严格匹配

# 大于1表示可以放大的倍数，小于1表示可以缩小的倍数

min_width = target_width / ratio_deviation

max_width = target_width * ratio_deviation + 2

min_height = target_height / ratio_deviation

max_height = target_height * ratio_deviation + 2

  

return (

min_width <= file_width <= max_width

and min_height <= file_height <= max_height

)

  

def _get_layer_config(

self, template: DynamicMaterialTemplate

) -> list[DimensionInfo]:

psd = SimplePSD.from_json(json.dumps(template.layout))

dynamic_layers = get_dynamic_layers(psd)

res = DimensionInfo.from_more_info(dynamic_layers, template.match_range)

return res

  

async def _match_layer_files(

self, file_path: str, template: DynamicMaterialTemplate

) -> list[DimensionInfo]:

base_path = template.input_path

real_base_path = self._get_real_path(base_path) if self.dry_run else base_path

  

dimension_infos = self._get_layer_config(template)

logger.info(f"real_base_path: {real_base_path}")

for root, _, files in os.walk(real_base_path):

logger.info(f"root: {root}, files: {files}")

for f in files:

if self.dry_run:

file = File(home=self.dry_run_prefix, path=os.path.join(root, f))

else:

file = File(home=self.home, path=base_path)

# if self.deduplicator and await self.deduplicator.is_dynamic_marked(

# file, template.id

# ):

# continue

  

dimensions = self._extract_dimensions_from_filename(f)

if not dimensions:

logger.debug(f"Could not extract dimensions from filename: {f}")

continue

  

file_width, file_height = dimensions

# 对每个图层检查文件是否匹配

for dim_info in dimension_infos:

if not dim_info.is_matches_filename(f):

logger.info(

f"file name {f} not contains any keyword in {dim_info.keywords}"

)

continue

if not self._is_size_in_range(

dim_info.layer_info.get("width", 0),

dim_info.layer_info.get("height", 0),

file_width,

file_height,

dim_info.ratio_deviation,

):

logger.debug(

f"File dimensions {file_width}x{file_height} not in range "

f"for layer {dim_info.name} ({dim_info.layer_info.get("width", 0)}x{dim_info.layer_info.get("height", 0)}) "

f"with ratio_deviation {dim_info.ratio_deviation}"

)

continue

  

full_path = os.path.join(root, f)

# if self.dry_run:

# full_path = self._remove_dry_run_prefix(full_path)

  

# 将匹配的文件添加到DimensionInfo中

dim_info.add_file(full_path)

logger.success(

f"Found matching file for layer: {dim_info.name}: {full_path} "

f"with dimensions {file_width}x{file_height}"

f"files: {dim_info.files}"

)

  

for dim_info in dimension_infos:

if not dim_info.has_matches:

logger.warning(

f"No matching files found for layer {dim_info.name} in path {base_path}"

)

  

return dimension_infos

  

def _validate_dimension_infos(

self,

list_dimension_infos: list[DimensionInfo],

template: DynamicMaterialTemplate,

) -> bool:

required_layers: set[str] = set()

  

def collect_dynamic_layers(node: dict[str, Any]) -> None:

if node.get("dynamic") and node.get("name") and node.get("visible"):

required_layers.add(node["name"])

for child in node.get("children", []):

collect_dynamic_layers(child)

  

collect_dynamic_layers(template.layout)

layer_info_map = {dim_info.name: dim_info for dim_info in list_dimension_infos}

  

# 检查每个必需的图层

for layer_name in required_layers:

dim_info = layer_info_map.get(layer_name)

if not dim_info:

logger.warning(

f"Required layer not found in dimension_infos: {layer_name}"

)

return False

  

if not dim_info.has_matches and dim_info.layer_info.get("visible"):

logger.warning(

f"Missing files for required layer: {layer_name}, "

f"width={dim_info.layer_info.get("width", 0)}, height={dim_info.layer_info.get("height", 0)}, "

f"ratio_deviation={dim_info.ratio_deviation}, "

f"keywords={dim_info.keywords}, ",

)

return False

  

logger.info(

f"Validated layer {layer_name} with {len(dim_info.files)} files: {dim_info.files}"

)

  

return True

  

async def _generate_dynamic_material(

self, file: File, template: DynamicMaterialTemplate

) -> tuple[bool, list[DimensionInfo] | None]:

try:

# 1. 匹配各个图层对应的文件

list_dimension_infos = await self._match_layer_files(file.path, template)

logger.with_field(list_dimension_infos=list_dimension_infos).info(

"list_dimension_infos"

)

# 2. 验证是否所有必需的图层都找到了对应的文件

if not self._validate_dimension_infos(list_dimension_infos, template):

logger.warn("Missing required layer files")

# return False, None

  

# 3. 确保输出目录存在

await self._ensure_output_path(template)

  

# 4. TODO: 根据layout配置和layer_files生成素材

logger.info("Now generate")

render = Render(list_dimension_infos)

  

res = await render.render(template)

for image in res:

image.save(f"{template.output_path}/{file.path.split('/')[-1]}.png")

return True, list_dimension_infos

  

except Exception as e:

logger.exception(f"Error generating dynamic material: {str(e)}")

return False, None

  

async def preprocess(

self, file: File, log: context.Context

) -> tuple[File | None, context.Context]:

return file, log

  

async def process(self, file: File, log: context.Context) -> bool:

if not self.deduplicator:

return False

  

async for template in self._get_template_for_path(file.path):

if not template:

continue

# 是否 重复生成

# 不同于最外层的重复，这里需要根据 template.id 来判断，避免不同文件根据同一配置生成重复素材（因为最外层的重复判断只针对单独文件，忽略了同一配置的其他文件）

# if await self.deduplicator.is_dynamic_marked(file, template.id):

# continue

# 确保输出目录存在

await self._ensure_output_path(template)

  

# TODO: 处理PSD动态素材生成逻辑

success, list_dimension_infos = await self._generate_dynamic_material(

file, template

)

  

if success and list_dimension_infos:

if self.deduplicator:

for dim_info in list_dimension_infos:

for file_path in dim_info.files:

file = File(home=self.home, path=file_path)

await self.deduplicator.mark_dynamic_template(

file, template.id

)

log.info(f"Successfully processed file {file.path}")

return True

log.error(f"Failed to process file {file.path}")

return False

  

return False

  
  

@coro("dsp")

async def main():

async with DynamicMaterial(

DynamicMaterialSetting(

dry_run=True,

dry_run_prefix="/Users/shianyu/Documents",

)

) as processor:

await processor.process(

file=File(

home="/Users/shianyu/documents",

path="/创意中心/物料库/动态素材/石安毓1/logo_taobao_389*23.png"

),

log=logger.with_field(file_id=1),

)

  
  

if __name__ == "__main__":

asyncio.run(main())
```
