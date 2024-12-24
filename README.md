# ChB_color_to_mat
We assign materials to the object based on the colors on the texture in the second UV channel.

```python
import bpy
import bmesh
import os
from mathutils import Vector

# Материалы для назначения
materials = {
    "green": "id100_r_land_02",
    "red": "id231_stn_03",
    "blue": "id55_sand_01",
    "default": "id165_wild_grass_5",
}

def get_udim_from_uv(uv):
    """
    Определяет номер UDIM из UV-координат второго UV-канала.
    """
    udim_base = 1001
    udim_x = int(uv.x)
    udim_y = int(uv.y)
    return udim_base + udim_x + (udim_y * 10)

def get_udim_from_texture_name(name):
    """
    Извлекает номер UDIM из имени текстуры, например: 1002_BC.
    """
    parts = name.split("_")
    if len(parts) > 0 and parts[0].isdigit():
        return int(parts[0])
    return None

def get_pixel_color(image, uv):
    """
    Получает цвет пикселя на текстуре для заданных UV-координат.
    """
    x = int(uv.x * image.size[0]) % image.size[0]
    y = int(uv.y * image.size[1]) % image.size[1]
    pixel_index = (y * image.size[0] + x) * 4
    return image.pixels[pixel_index:pixel_index + 3]  # Только RGB

def assign_materials_to_mesh(mesh_obj, images_by_udim, material_map):
    """
    Назначает материалы полигонам меша на основе текстур и UV-координат из второго UV-канала.
    """
    bm = bmesh.new()
    bm.from_mesh(mesh_obj.data)

    uv_layer = bm.loops.layers.uv.get("UVChannel_2")
    if not uv_layer:
        print(f"UVChannel_2 не найден на меше {mesh_obj.name}.")
        bm.free()
        return

    # Построим карту индексов материалов для быстрого доступа
    material_index_map = {key: mesh_obj.data.materials.find(mat.name) for key, mat in material_map.items()}

    # Проверка каждого полигона
    for face in bm.faces:
        center_uv = Vector((0.0, 0.0))
        for loop in face.loops:
            center_uv += loop[uv_layer].uv
        center_uv /= len(face.loops)

        udim = get_udim_from_uv(center_uv)
        image = images_by_udim.get(udim)

        if not image:
            print(f"Текстура для UDIM {udim} не найдена. Пропуск полигона.")
            continue

        # Получаем цвет пикселя в центре полигона
        color = get_pixel_color(image, center_uv)
        print(f"Объект: {mesh_obj.name}, UDIM: {udim}, Центр UV: {center_uv}, Цвет: {color}")

        # Определяем материал
        if color[0] > 0.5 and color[1] < 0.5 and color[2] < 0.5:
            material_key = "red"
        elif color[0] < 0.5 and color[1] > 0.5 and color[2] < 0.5:
            material_key = "green"
        elif color[0] < 0.5 and color[1] < 0.5 and color[2] > 0.5:
            material_key = "blue"
        else:
            material_key = "default"

        # Назначаем материал полигону
        if material_index_map[material_key] != -1:
            face.material_index = material_index_map[material_key]
        else:
            print(f"Материал '{material_key}' не найден для объекта {mesh_obj.name}.")

    # Применяем изменения
    bm.to_mesh(mesh_obj.data)
    bm.free()

def assign_textures_to_meshes_from_folder(folder_path):
    """
    Назначает текстуры на основе UDIM для выделенных мешей.
    """
    if not os.path.exists(folder_path):
        print(f"Папка {folder_path} не найдена.")
        return

    # Собираем текстуры из папки
    texture_files = [f for f in os.listdir(folder_path) if f.lower().endswith(('.png', '.jpg', '.bmp'))]

    # Сопоставление UDIM текстур
    images_by_udim = {}
    for texture_file in texture_files:
        udim = get_udim_from_texture_name(texture_file)
        if udim:
            texture_path = os.path.join(folder_path, texture_file)
            try:
                image = bpy.data.images.load(texture_path, check_existing=True)
                images_by_udim[udim] = image
                print(f"Загружена текстура {texture_path} для UDIM {udim}.")
            except RuntimeError:
                print(f"Ошибка загрузки текстуры: {texture_path}")

    # Обработка каждого выделенного объекта
    for obj in bpy.context.selected_objects:
        if obj.type != 'MESH':
            continue

        # Создаём или получаем материалы назначения
        material_map = {}
        for key, mat_name in materials.items():
            mat = bpy.data.materials.get(mat_name)
            if not mat:
                mat = bpy.data.materials.new(name=mat_name)
            if mat.name not in [m.name for m in obj.data.materials]:
                obj.data.materials.append(mat)
            material_map[key] = mat

        # Назначаем материалы полигонам
        assign_materials_to_mesh(obj, images_by_udim, material_map)

# Путь к папке с текстурами
texture_folder_path = r"D:\3D\Work_project\ChillBAse\Country Side Beach\sp\Textur\Color_ID"

# Запуск функции
assign_textures_to_meshes_from_folder(texture_folder_path)
