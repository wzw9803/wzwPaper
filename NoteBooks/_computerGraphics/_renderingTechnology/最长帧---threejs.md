# 最长帧---threejs

## 1. threejs内部自增 _object3DId 初始化后是5，以下分析引擎内部在初始化时自己创建了哪些 Object3D 对象

1. _object3DId： 0
2. _obj： 1, class: Object3D (BufferGeometry的工具)
3. _mesh： 2, class: Mesh (InstancedMesh的工具)
4. _obj$1：3, class: Object3D (Geometry的工具)
5. _camera: 4, class: Camera (CameraHelper的工具)
6. _flatCamera: 5 ,class: OrthographicCamera (PMREMGenerator的工具)

自此 _object3DId 自增为5，_object3DId 5以后才是用户创建的Object3D的对象。

