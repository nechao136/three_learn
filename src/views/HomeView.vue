<template>
  <div ref="container" class="home">

  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { VRM, VRMLoaderPlugin } from '@pixiv/three-vrm'


enum AnimationState {
  IDLE,
  SPEAKING,
}
@Component({})
export default class HomeView extends Vue {
  private scene!: THREE.Scene
  private camera!: THREE.PerspectiveCamera
  private renderer!: THREE.WebGLRenderer
  private controls!: OrbitControls
  private loader!: GLTFLoader
  private vrm!: VRM
  private clock!: THREE.Clock
  private state: AnimationState = AnimationState.IDLE

  private randomIdleMotion(vrm: VRM) {
    // 头部微微摇动
    const head = vrm.humanoid.getNormalizedBoneNode('head') ?? new THREE.Object3D()
    head.rotation.y = Math.sin(Date.now() * 0.001) * 0.1
    head.rotation.x = Math.sin(Date.now() * 0.0015) * 0.05

    // 手臂轻微摆动
    const leftUpperArm = vrm.humanoid.getNormalizedBoneNode('leftUpperArm') ?? new THREE.Object3D()
    const rightUpperArm = vrm.humanoid.getNormalizedBoneNode('rightUpperArm') ?? new THREE.Object3D()
    leftUpperArm.rotation.z = 1.2 + Math.sin(Date.now() * 0.002) * 0.05
    rightUpperArm.rotation.z = -1.2 + Math.sin(Date.now() * 0.0025) * 0.05

    // 随机眨眼
    const em = vrm.expressionManager
    if (Math.random() < 0.005) em?.setValue('blink', 1)
    else em?.setValue('blink', 0)
  }
  private speakingMotion(vrm: VRM) {
    const em = vrm.expressionManager
    // 模拟嘴巴开合
    em?.setValue('aa', Math.random())
  }
  private animate() {
    requestAnimationFrame(this.animate);
    const delta = this.clock.getDelta()
    if (!!this.vrm) {
      // 更新物理效果（头发、裙子等）
      this.vrm.update(delta)
      if (this.state === AnimationState.IDLE) {
        this.randomIdleMotion(this.vrm)
      }
      else if (this.state === AnimationState.SPEAKING) {
        this.speakingMotion(this.vrm)
      }
    }
    this.renderer.render(this.scene, this.camera)
  }
  private resizeEvent() {
    this.$nextTick(() => {
      const container = this.$refs?.['container'] as HTMLDivElement
      const width = container.clientWidth
      const height = container.clientHeight

      this.camera.aspect = width / height
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(width, height)
    })
  }

  protected async created() {
    this.$nextTick(() => {
      const container = this.$refs?.['container'] as HTMLDivElement
      const width = container.clientWidth
      const height = container.clientHeight

      // 1. 创建场景
      this.scene = new THREE.Scene()

      // 2. 摄像机
      this.camera = new THREE.PerspectiveCamera(40, width / height, 0.1, 1000)
      this.camera.position.set(0, 1.5, -3); // Z 轴负方向
      this.camera.lookAt(0, 1, 0); // 目标指向模型中心

      // 3. 渲染器
      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
      })
      this.renderer.setSize(width, height)
      this.renderer.setClearColor(0xf0f0f0, 1)
      container.appendChild(this.renderer.domElement)

      // 4. 光源
      // 环境光，均匀照亮场景
      const ambient = new THREE.AmbientLight(0xffffff, 0.6)
      this.scene.add(ambient)
      // 主光源（模拟太阳）
      const directional = new THREE.DirectionalLight(0xffffff, 0.8)
      directional.position.set(1, 2, 2)
      this.scene.add(directional)
      // 补光（柔化阴影）
      const fillLight = new THREE.DirectionalLight(0xffffff, 0.3)
      fillLight.position.set(-1, 0.5, 1)
      this.scene.add(fillLight)

      // 5. 控制器
      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.target.set(0, 1, 0); // 让模型中心为旋转中心
      this.controls.maxPolarAngle = Math.PI / 2; // 限制旋转角度
      this.controls.minDistance = 1.5;
      this.controls.maxDistance = 5;
      this.controls.update();

      // 6. 加载 VRM 模型
      this.loader = new GLTFLoader()
      this.loader.register(parser => new VRMLoaderPlugin(parser))
      this.clock = new THREE.Clock()
      this.loader.load(
        '/model/avex.vrm',
        async (gltf) => {
          this.vrm = gltf.userData?.['vrm']
          this.scene.add(this.vrm.scene)
          console.log('VRM loaded:', this.vrm)
        },
        (progress) => console.log('Loading VRM...', 100.0 * (progress.loaded / progress.total), '%'),
        (error) => console.error('VRM loading error:', error))

      // 7. 渲染循环
      this.animate()
    })
    window.addEventListener('resize', this.resizeEvent)
  }
  protected async beforeDestroy() {
    window.removeEventListener('resize', this.resizeEvent)
  }
}
</script>
<style lang="scss" scoped>
.home {
  width: 100vw;
  height: 100vh;
}
</style>
