<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import LoadingIco from './components/LoadingIco.vue'
import PageWrap from './components/PageWrap.vue'
import Header from './components/Header.vue'
import FooterInfo from './components/FooterInfo.vue'
import atmosphereVertexShader from './shaders/atmosphere/vertex.glsl?raw'
import atmosphereFragmentShader from './shaders/atmosphere/fragment.glsl?raw'

// Refs
const webgl = ref(null)

// Scene
const scene = new THREE.Scene()
scene.background = new THREE.Color('#99a1af')

// Texture Loaders
const textureLoader = new THREE.TextureLoader()

// Resize
const sizes = {
    width: window.innerWidth,
    height: window.innerHeight,
    pixelRatio: Math.min(window.devicePixelRatio, 2)
}

const handleResize = () => {
    // Update sizes
    sizes.width = window.innerWidth
    sizes.height = window.innerHeight
    sizes.pixelRatio = Math.min(window.devicePixelRatio, 2)

    // Update camera
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()

    // Update renderer
    renderer.setSize(sizes.width, sizes.height)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
}

// Loading Manager
const loading = ref(true)
const loadingManager = new THREE.LoadingManager(
    () => {
        loading.value = false
    },
    (file, loaded, total) => {
        const progress = loaded / total
        console.log(`Loading: ${progress * 100}%`)
    }
)

// Renderer
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
renderer.toneMapping = THREE.ACESFilmicToneMapping
renderer.outputEncoding = THREE.sRGBEncoding
renderer.physicallyCorrectLights = true
renderer.shadowMap.enabled = true
renderer.shadowMap.type = THREE.PCFSoftShadowMap

// Sun Light
const sunLight = new THREE.DirectionalLight(0xffffff, 5.5)
sunLight.position.set(10, 20, 10)
sunLight.castShadow = true
sunLight.shadow.mapSize.width = 512
sunLight.shadow.mapSize.height = 512
sunLight.shadow.camera.far = 100
sunLight.shadow.camera.near = 0.5
sunLight.shadow.camera.left = -10
sunLight.shadow.camera.right = 10
sunLight.shadow.camera.top = 10
sunLight.shadow.camera.bottom = -10
scene.add(sunLight)

// Camera
const camera = new THREE.PerspectiveCamera(45, sizes.width / sizes.height, 0.1, 1000)
camera.position.set(0, 15, 50)
scene.add(camera)

// Controls
const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true
controls.enablePan = false
controls.enableZoom = false
// controls.autoRotate = true
// controls.autoRotateSpeed = 0.5

// Scene Setup
const setupScene = () => {
    // HDR Equirectangular
    const hdrEquirect = new RGBELoader(loadingManager).load(
        './textures/brown_photostudio_04_1k.hdr',
        function (texture) {
            texture.mapping = THREE.EquirectangularReflectionMapping
        }
    )

    // Textures
    let textures = {
        bump: textureLoader.load('./textures/earthbump.jpg'),
        map: textureLoader.load('./textures/earthmap.jpg'),
        spec: textureLoader.load('./textures/earthspec.jpg')
    }

    // Earth
    let sphere = new THREE.Mesh(
        new THREE.SphereGeometry(10, 70, 70),
        new THREE.MeshPhysicalMaterial({
            map: textures.map,
            bumpMap: textures.bump,
            bumpScale: 0.05,
            roughnessMap: textures.spec,
            envMap: hdrEquirect,
            envMapIntensity: 0.6,
            sheen: 0.5,
            sheenRoughness: 0.8,
            sheenColor: new THREE.Color('#efefef').convertSRGBToLinear(),
            clearcoat: 0.5
        })
    )
    sphere.rotation.y += Math.PI * 0.85
    sphere.receiveShadow = true
    scene.add(sphere)

    const earthParameters = {}
    earthParameters.atmosphereDayColor = '#afa1ff'
    earthParameters.atmosphereTwilightColor = '#ff6600'

    // Atmosphere
    const atmosphereMaterial = new THREE.ShaderMaterial({
        vertexShader: atmosphereVertexShader,
        fragmentShader: atmosphereFragmentShader,
        uniforms: {
            uSunDirection: new THREE.Uniform(new THREE.Vector3(0, 0, 3)),
            uAtmosphereDayColor: new THREE.Uniform(new THREE.Color(earthParameters.atmosphereDayColor)),
            uAtmosphereTwilightColor: new THREE.Uniform(new THREE.Color(earthParameters.atmosphereTwilightColor))
        },
        side: THREE.BackSide,
        transparent: true
    })
    const atmosphere = new THREE.Mesh(sphere.geometry, atmosphereMaterial)
    atmosphere.scale.set(1.04, 1.04, 1.04)
    scene.add(atmosphere)

    // Plane
    const loader = new GLTFLoader()

    let plane

    loader.load('./textures/plane.glb', (gltf) => {
        scene.add(gltf.scene)
        plane = gltf.scene.children[0] // 假設 plane.glb 的根節點是 scene，且第一個子節點是 plane
        plane.scale.set(0.001, 0.001, 0.001)
        plane.position.set(0, 0, 0)
        plane.rotation.set(0, 0, 0)
        plane.updateMatrixWorld()

        plane.traverse((object) => {
            if (object instanceof THREE.Mesh) {
                object.material.envMap = hdrEquirect
                object.castShadow = true
                object.receiveShadow = true
            }
        })
    })

    // Animate
    const clock = new THREE.Clock()

    const tick = () => {
        const elapsedTime = clock.getElapsedTime()

        sphere.rotation.y = elapsedTime * 0.1

        // Make the plane rotate around the sphere
        if (plane) {
            const radius = 11 // Adjust the radius as needed
            const angle = elapsedTime * 0.7 // Adjust the rotation speed as needed
            plane.position.x = Math.cos(angle) * radius
            plane.position.z = Math.sin(angle) * radius
            plane.position.y = 2.5 // Adjust the height as needed
            plane.updateMatrixWorld()
            plane.lookAt(sphere.position)
            plane.rotateZ(Math.PI / -2)
        }

        // Render
        renderer.render(scene, camera)
        controls.update()

        // Call tick again on the next frame
        window.requestAnimationFrame(tick)
    }

    tick()

    renderer.render(scene, camera)

    window.requestAnimationFrame(tick)
}

onMounted(() => {
    webgl.value.appendChild(renderer.domElement)
    window.addEventListener('resize', handleResize)

    setupScene()
})
</script>

<template>
    <PageWrap>
        <Header />
        <div v-if="loading" class="z-10 h-dvh inset-0 flex items-center justify-center">
            <LoadingIco />
        </div>
        <div class="outline-none w-full h-dvh" ref="webgl"></div>

        <FooterInfo>
            <template v-slot:title></template>
            <template v-slot:first>
                <a
                    href="https://www.youtube.com/watch?v=xECyFOsVZ0E&list=WL&index=4&t=351s"
                    target="_blank"
                    class="underline-offset-2 font-medium"
                >
                    @Irradiance Earth and planes - tutorial</a
                >
            </template>
            <template v-slot:second> </template>
            <template v-slot:github>
                <a href="https://pccathome.github.io/fly-around-the-world/" target="_blank" class="underline-offset-2"
                    >GitHub</a
                >
            </template>
        </FooterInfo>
    </PageWrap>
</template>
