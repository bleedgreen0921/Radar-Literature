# ROS2使用

## Ubuntu环境下C++文件的编写

1. **环境准备**

   ```bash
   # 更新系统
   sudo apt update && sudo apt upgrade -y
   
   # 安装ROS 2 Humble（若未安装）
   sudo apt install ros-humble-desktop ros-dev-tools
   
   # 安装编译工具
   sudo apt install build-essential cmake python3-colcon-common-extensions
   
   # 配置环境变量
   echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
   source ~/.bashrc
   ```

2. **创建工作空间**

   ```bash
   mkdir -p ~/ros2_ws/src
   cd ~/ros2_ws
   colcon build  # 初始化工作空间
   ```

3. **创建C++ ROS 2包**

   ```bash
   cd ~/ros2_ws/src
   ros2 pkg create my_cpp_pkg --build-type ament_cmake --dependencies rclcpp
   ```

4. **编写C++节点**

   1. **创建节点文件**

      ```bash
      mkdir -p ~/ros2_ws/src/my_cpp_pkg/src
      nano ~/ros2_ws/src/my_cpp_pkg/src/my_node.cpp
      ```

   2. **写入代码**

      ```cpp
      #include "rclcpp/rclcpp.hpp" // 注意导入头文件
      ```

5. **配置编译系统**

   1. **修改 CMakeLists.txt**

      ```bash
      nano ~/ros2_ws/src/my_cpp_pkg/CMakeLists.txt
      // 在文件末尾添加
      add_executable(my_cpp_node src/my_node.cpp)
      ament_target_dependencies(my_cpp_node rclcpp)
      install(TARGETS my_cpp_node DESTINATION lib/${PROJECT_NAME})
      ```

   2. **检查 package.xml**

      ```xml
      <depend>rclcpp</depend> // 检查是否含有
      ```

6. **编译工作空间**

   ```bash
   cd ~/ros2_ws
   colcon build --symlink-install  # --symlink-install允许开发时实时修改生效
   source install/setup.bash
   ```

7. **运行节点**

   ```bash
   ros2 run my_cpp_pkg my_cpp_node
   ```

## Ubuntu环境下Python文件的编写

1. **环境准备**

   ```bash
   # 更新系统
   sudo apt update && sudo apt upgrade -y
   
   # 安装 ROS 2 Humble（如果尚未安装）
   sudo apt install ros-humble-desktop python3-colcon-common-extensions
   
   # 配置环境变量
   echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
   source ~/.bashrc
   ```

2. **创建工作空间**

   ```bash
   mkdir -p ~/ros2_ws/src
   cd ~/ros2_ws
   colcon build  # 初始化工作空间
   ```

3. **创建Python Ros2包**

   ```bash
   cd ~/ros2_ws/src
   ros2 pkg create my_py_pkg --build-type ament_python --dependencies rclpy
   ```

4. **编写 Python 节点**

   1. **创建节点文件**

      ```bash
      touch ~/ros2_ws/src/my_py_pkg/my_py_pkg/my_node.py
      ```

   2. **编写代码**

      ```python
      #!/usr/bin/env python3
      import rclpy
      from rclpy.node import Node
      ...... # 代码主体
      if __name__ == '__main__':
          main()
      ```

   3. **设置可执行权限**

      ```bash
      chmod +x ~/ros2_ws/src/my_py_pkg/my_py_pkg/my_node.py
      ```

5. **配置包信息**

   1. 修改setup.py

      ```bash
      nano ~/ros2_ws/src/my_py_pkg/setup.py
      #更改entry_points部分：
      entry_points={
          'console_scripts': [
              'my_py_node = my_py_pkg.my_node:main',
          ],
      },
      ```

   2. 检查package.xml

      ```python
      # 确保包含如下代码：
      <exec_depend>rclpy</exec_depend>
      ```

6. **编译工作空间**

   ```bash
   cd ~/ros2_ws
   colcon build --symlink-install  # --symlink-install 允许开发时实时修改生效
   source install/setup.bash
   ```

7. **运行节点**

   ```bash
   ros2 run my_py_pkg my_py_node
   ```

## ubuntu环境下使用vscode进行ros2项目创建和编程

1. **创建ROS 2 工作空间**

   ```bash
   mkdir -p ~/ros2_ws/src
   cd ~/ros2_ws
   colcon build
   echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
   source ~/.bashrc
   ```

2. **在VS Code中打开项目**

   ```bash
   code ~/ros2_ws
   ```

3. **手动添加配置文件**

   ```bash
   # 进入工作空间 src 目录
   cd ~/ros2_ws/src
   
   # 创建 C++ 包
   ros2 pkg create my_pkg --build-type ament_cmake --dependencies rclcpp
   
   # 创建 .vscode/c_cpp_properties.json：
   {
     "configurations": [
       {
         "name": "Linux",
         "includePath": [
           "/opt/ros/humble/include/**",
           "${workspaceFolder}/**"
         ],
         "defines": [],
         "compilerPath": "/usr/bin/gcc",
         "cStandard": "gnu17",
         "cppStandard": "gnu++14",
         "intelliSenseMode": "linux-gcc-x64"
       }
     ],
     "version": 4
   }
   ```

4. **开发C++节点**

   1. **创建节点文件**

      ```cpp
      #include "rclcpp/rclcpp.hpp"
      int main(){
          ......
              return 0;
      }
      ```

   2. **修改 CMakeLists.txt**

      ```cmake
      // 在add_executable部分添加：
      add_executable(minimal_node src/minimal_node.cpp)
      ament_target_dependencies(minimal_node rclcpp)
      install(TARGETS minimal_node DESTINATION lib/${PROJECT_NAME})
      ```

5. **编译与运行**

   ```bash
   # 编译
   colcon build --symlink-install
   
   # 运行节点
   source install/setup.bash
   ros2 run my_pkg minimal_node
   
   # 调试配置
   {
     "version": "0.2.0",
     "configurations": [
       {
         "name": "ROS2 Debug",
         "type": "cppdbg",
         "request": "launch",
         "program": "${workspaceFolder}/install/my_pkg/lib/my_pkg/minimal_node",
         "args": [],
         "cwd": "${workspaceFolder}",
         "environment": [
           {"name": "LD_LIBRARY_PATH", "value": "/opt/ros/humble/lib:${env:LD_LIBRARY_PATH}"}
         ]
       }
     ]
   }
   ```

6. **高级配置**

   1. **自动补全优化**

      ```json
      // 在 .vscode/c_cpp_properties.json 中添加
      {
        "configurations": [
          {
            "includePath": [
              "/opt/ros/humble/include/**",
              "${workspaceFolder}/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc"
          }
        ]
      }
      ```

   2. **任务自动化**

      ```json
      // 在 .vscode/tasks.json 中添加
      {
        "label": "colcon build",
        "command": "colcon",
        "args": ["build", "--symlink-install"],
        "options": {
          "cwd": "${workspaceFolder}"
        }
      }
      ```

## ubuntu环境下怎么使用vscode的Python插件进行ros2项目创建和编程

1. **创建ROS 2 Python 工作空间**

   ```bash
   mkdir -p ~/ros2_ws/src
   cd ~/ros2_ws
   colcon build
   echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
   source ~/.bashrc
   ```

2. **通过命令行创建Python包**

   ```bash
   cd ~/ros2_ws/src
   ros2 pkg create my_py_pkg --build-type ament_python --dependencies rclpy
   ```

3. **开发python节点**

   1. **创建节点文件**

      ```python
      # 在 my_py_pkg/my_py_pkg/ 下新建 minimal_node.py
      #!/usr/bin/env python3
      import rclpy
      from rclpy.node import Node
      ......
      if __name__ == '__main__':
          main()
      ```

   2. **配置setup.py**

      ```python
      # 修改 entry_points 部分
      entry_points={
          'console_scripts': [
              'minimal_node = my_py_pkg.minimal_node:main',
          ],
      },
      ```

   3. **设置可执行权限**

      ```bash
      chmod +x ~/ros2_ws/src/my_py_pkg/my_py_pkg/minimal_node.py
      ```

4. **VS Code专项配置**

   1. **选择Python解释器**

   2. **配置调试**

      ```json
      // 创建 .vscode/launch.json
      {
        "version": "0.2.0",
        "configurations": [
          {
            "name": "Python: ROS2 Node",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/src/my_py_pkg/my_py_pkg/minimal_node.py",
            "console": "integratedTerminal",
            "justMyCode": false,
            "env": {
              "PYTHONPATH": "${workspaceFolder}/install/my_py_pkg/lib/python3.10/site-packages:${env:PYTHONPATH}"
            }
          }
        ]
      }
      ```

   3. **启用自动补全**

      ```json
      // 在 .vscode/settings.json 中添加
      {
        "python.analysis.extraPaths": [
          "/opt/ros/humble/lib/python3.10/site-packages"
        ]
      }
      ```

5. **编译与运行**

   ```bash
   cd ~/ros2_ws
   colcon build --symlink-install
   source install/setup.bash
   ros2 run my_py_pkg minimal_node
   ```

