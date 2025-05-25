# 🛰️ SIBGRAPI2025_SLAM

Projeto para reconstrução 3D com base em imagens monoculares utilizando estimativa de profundidade e geração de nuvens de pontos integradas ao OctoMap via ROS 2 Humble (Ubuntu 22.04).

---

## ✨ Visão Geral

Este sistema realiza:

- Estimativa de profundidade com **DepthAnythingV2**.
- Geração e visualização de nuvens de pontos com **Open3D**.
- Publicação das nuvens no ROS 2 usando **PointCloud2**.
- Mapeamento 3D com **`octomap_server`** e visualização no **RViz**.

---

## ⚙️ Etapas de Instalação

### 1. Criar workspace ROS 2

```bash
mkdir -p ~/octomap_ws/src
cd ~/octomap_ws
colcon build
source install/setup.bash
echo "source ~/octomap_ws/install/setup.bash" >> ~/.bashrc
```

### 2. Instalar pacotes do OctoMap

```bash
sudo apt update
sudo apt install ros-humble-octomap ros-humble-octomap-msgs ros-humble-octomap-server
```

---

## 🧱 Construção do Publicador ROS

### 3. Criar pacote Python `o3d_publisher`

```bash
cd ~/octomap_ws/src
ros2 pkg create --build-type ament_python o3d_publisher --dependencies rclpy sensor_msgs std_msgs
```

Edite `setup.py` do pacote com os blocos `data_files` e `entry_points`:

```python
    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        ('share/' + package_name + '/launch', ['launch/octomap_launch.py']),
    ],

    entry_points={
        'console_scripts': [
            'o3d_pub_node = o3d_publisher.o3d_pub_node:main',
        ],
    },
```

### 4. Criar o script `o3d_pub_node.py`

Responsável por carregar, converter e publicar nuvens de pontos `.ply` com Open3D.  
```bash
mkdir -p ~/octomap_ws/src/o3d_publisher/o3d_publisher
nano ~/octomap_ws/src/o3d_publisher/o3d_publisher/o3d_pub_node.py
```

```python
import rclpy
from rclpy.node import Node

from sensor_msgs.msg import PointCloud2, PointField
from std_msgs.msg import Header

import numpy as np
import struct

import open3d as o3d


def convert_cloud_to_ros_msg(cloud: o3d.geometry.PointCloud, stamp, frame_id="map") -> PointCloud2:
    points = np.asarray(cloud.points)
    colors = np.asarray(cloud.colors)

    if colors.shape[0] != points.shape[0]:
        colors = np.zeros_like(points)

    data = []
    for i in range(points.shape[0]):
        x, y, z = points[i]
        r, g, b = (colors[i] * 255).astype(np.uint8)
        rgb = struct.unpack('I', struct.pack('BBBB', b, g, r, 0))[0]
        data.append([x, y, z, rgb])
    data = np.array(data, dtype=np.float32)

    msg = PointCloud2()
    msg.header.stamp = stamp
    msg.header.frame_id = frame_id
    msg.height = 1
    msg.width = points.shape[0]
    msg.fields = [
        PointField(name='x', offset=0, datatype=PointField.FLOAT32, count=1),
        PointField(name='y', offset=4, datatype=PointField.FLOAT32, count=1),
        PointField(name='z', offset=8, datatype=PointField.FLOAT32, count=1),
        PointField(name='rgb', offset=12, datatype=PointField.UINT32, count=1)
    ]
    msg.is_bigendian = False
    msg.point_step = 16
    msg.row_step = msg.point_step * points.shape[0]
    msg.is_dense = True
    msg.data = data.tobytes()
    return msg


class O3DPublisher(Node):
    def __init__(self):
        super().__init__('o3d_publisher')
        self.publisher_ = self.create_publisher(PointCloud2, 'o3d_points', 10)
        self.timer = self.create_timer(1.0, self.timer_callback)  # 1 Hz

        self.get_logger().info('Open3D publisher ready.')

    def timer_callback(self):
        try:
            pcd = o3d.io.read_point_cloud("/home/werikson/octomap_ws/src/SIBGRAPI2025_slam/point_clouds/pcd.ply") # Mude para o caminho dos seus dados
            stamp = self.get_clock().now().to_msg()
            msg = convert_cloud_to_ros_msg(pcd, stamp)
            self.publisher_.publish(msg)
            self.get_logger().info('Nuvem publicada!')
        except Exception as e:
            self.get_logger().error(f"Erro ao publicar nuvem: {e}")


def main(args=None):
    rclpy.init(args=args)
    node = O3DPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Salve e feche o arquivo (Ctrl+O, Enter, Ctrl+X).

### 5. Criar o arquivo de launch `octomap_launch.py`

```bash
mkdir -p ~/octomap_ws/src/o3d_publisher/launch
nano ~/octomap_ws/src/o3d_publisher/launch/octomap_launch.py
```

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='octomap_server',
            executable='octomap_server_node',
            name='octomap_server',
            output='screen',
            parameters=[{
                'frame_id': 'map',
                'resolution': 0.05,
                'sensor_model_max_range': 5.0
            }],
            remappings=[
                ('cloud_in', 'o3d_points')
            ]
        )
    ])
```

### 6. Compilar e ativar o ambiente

```bash
cd ~/octomap_ws
colcon build --packages-select o3d_publisher
source install/setup.bash
```

---

## 🧪 Execução do Sistema

### 7. Publicar com ROS:

```bash
ros2 run o3d_publisher o3d_pub_node
```

### 8. Rodar o servidor Octomap:

```bash
ros2 launch o3d_publisher octomap_launch.py
```

### 9. Visualizar no RViz:

```bash
rviz2
```

- **Fix Frame:** `map`
- **Add:** `/occupied_cells_vis_array` (`MarkerArray`)

### Monitorar GPU:

```bash
watch -n 1 nvidia-smi --id=0
```

---

## ✅ Resultado Esperado

Com os nós corretamente conectados:

* A nuvem de pontos será publicada no tópico `/o3d_points`.
* O `octomap_server` irá construir e manter o mapa 3D dinâmico.
* O resultado será visualizado em tempo real no RViz.

---
