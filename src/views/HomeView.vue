<template>
  <canvas ref="canvasRef"></canvas>
</template>

<script setup>
import { ref, onMounted, onUnmounted} from "vue"
import { OrbitControls} from "three/examples/jsm/controls/OrbitControls"
import * as THREE from "three"
import gsap from "gsap"
import * as dat from "dat.gui"


// Inizializziamo la GUI (Puoi nascondere il pannello premendo H sulla tastiera)
const gui = new dat.GUI()


let canvasRef = ref();

var controls

//Utilizziamo i valori del viewport per impostare la grandezza del canvas
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

//Aggiungiamo l'event listner per il resize del canvas
window.addEventListener("resize", ()=>{
  //update sizes for the canvas
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  //update the camera aspect ratio
  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  //update renderer
  renderer.setSize(sizes.width, sizes.height)
  //HANDLE PIXEL RATIO
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})


//HANDLE FULL SCREEN (Il codice aggiuntivo è per permettere il funzionamento corretto anche con browser Safari)
window.addEventListener("dblclick",()=>{
  const fullscreenElement = document.fullscreenElement ||  document.webkitFullscreenElement
  if(!fullscreenElement){
    if(canvasRef.value.requestFullscreen){
      canvasRef.value.requestFullscreen()
    }
    else if(canvasRef.value.webkitRequestFullscreen){
      canvasRef.value.webkitRequestFullscreen()
    }
    
  }
  else{
    if(document.exitFullscreen)
    {
      document.exitFullscreen()
    }
    else if(document.webkitExitFullscreen){
      document.webkitExitFullscreen()
    }
  }
})

let scene = new THREE.Scene()

let renderer

//TEXTURES
const textureLoader = new THREE.TextureLoader()
const bakeShadow = textureLoader.load("/textures/bakedShadow.jpg")
const simpleShadow = textureLoader.load("/textures/simpleShadow.jpg")

//  AMBIENT LIGHT
const ambientLight = new THREE.AmbientLight(0xffffff,0.3)
gui.add(ambientLight, "intensity").min(0).max(1).step(0.001)
scene.add(ambientLight)

// DIRECTIONAL LIGHT
const directionalLight = new THREE.DirectionalLight(0xffffff, 0.3)
directionalLight.position.set(2, 2, -1)
directionalLight.castShadow = true
gui.add(directionalLight, "intensity").min(0).max(1).step(0.001)
gui.add(directionalLight.position, "x").min(-5).max(5).step(0.001)
gui.add(directionalLight.position, "y").min(-5).max(5).step(0.001)
gui.add(directionalLight.position, "z").min(-5).max(5).step(0.001)
scene.add(directionalLight)

//SHADOWS
// first of all, we need to check in the render, and enable the shadowMap

//Then go in each of the objects and dedice if it can cast (object.castShadow = true) or receive shadow (object.receiveShadow = true)

//and finally go where you instantiate the light source and activate the shadow on that source of light

//NOTE that only SpotLight, PointLight and DirectionalLight support shadows

//OPTIMIZE THE SHADOW
//the shadow is not good in terms of quality, so we need to optimize it. 
// first we can change the shadowMap renderer size. by default it's 512 x 512 
// we can improve it , (keeping a power of the for the mipmapping)

directionalLight.shadow.mapSize.width = 1024
directionalLight.shadow.mapSize.height = 1024

//because we are doing a render, we have access also to the near and the far of that camera
//so we can just move the near and the far , in a way it fits the scene
//this won't improve your result, but you need it to avoid glitches in the shadow
//(note that spotlight has perspective camera, and directional light has ortographic camera)

// so now we can tweek the near and far parameters
directionalLight.shadow.camera.near = 1
directionalLight.shadow.camera.far = 6


//Next is about the amplitude. Now it's doing a huge render that is bigger than the scene
//so we can reduce the amplitude. and because this it's and ortographic camera we can tweek also the left ,top, right, bottom
directionalLight.shadow.camera.top = 2
directionalLight.shadow.camera.right = 2
directionalLight.shadow.camera.bottom = -2
directionalLight.shadow.camera.left = -2


//BLUR of the Shadow

// We have also control on the blur of the shadow, using the radius property (this don't use the proximity of the camera with the object. it's a cheap and general blur)
directionalLight.shadow.radius = 10


//but we can't see this camera, so we are going to use another helper for visualize the camera
const directionalLightCameraHelper = new THREE.CameraHelper(directionalLight.shadow.camera)
directionalLightCameraHelper.visible = false
scene.add(directionalLightCameraHelper)


//Shadow Map Algorithm
//There are different types of algorithm that can be applied to the shadow maps. the most used are:
// THREE.BasicShadowMap (most performant but lously quality)
// THREE.PCFShadowMap (the default. less performant but smoother edges.)
// THREE.PCESoftShadowMap (less performant but more softer edges)
// THREE.VSMShadowMap (less performant, hard to implement, more constrains, can have unexpected result)

//we can set this, down where we instantiate the renderer
//NOTE that the radius doesn't work with THREE.PCFSoftShadowMap



// SHADOW WITH SpotLight
const spotLight = new THREE.SpotLight(0xffffff, 0.3, 10, Math.PI * 0.3)
spotLight.castShadow = true
spotLight.position.set(0, 2, 2)
scene.add(spotLight)
scene.add(spotLight.target)


//like we did for the directionalLight , first we need to tweek the size of the spotlight camera 
spotLight.shadow.mapSize.width = 1024
spotLight.shadow.mapSize.height = 1024

//then, because we are using a spotLight, the camera type is Perspective, so for adapt the amplitude we must tweek the "fov" property of the camera
spotLight.shadow.camera.fov = 30

//and finally we can change the near and the far property value
spotLight.shadow.camera.near = 1
spotLight.shadow.camera.far = 6

//for visualize the size of the shadow camera use the camera helper
const spotLightCameraHelper = new THREE.CameraHelper(spotLight.shadow.camera)
scene.add(spotLightCameraHelper)
spotLightCameraHelper.visible = false




//SHADOW WITH Point Light
const pointLight = new THREE.PointLight(0xffffff, 0.4)
pointLight.castShadow = true

pointLight.position.set(-1, 1, 0)
scene.add(pointLight)

//with the pointlight the light are spread in every direction, so when the render goes in, it takes a "pictures" of all of the 6 face of the scene (top, bottom, forward, back, left, right), and it the reason why we see the camera facing down, because is the last step

//we can tweak the mapsize, the near and far (we cannot control the field of view, because it's used to render all the surrounding, if we reduce this, it will cause issue)
pointLight.shadow.mapSize.width = 1024
pointLight.shadow.mapSize.height = 1024

pointLight.shadow.camera.near = 0.1
pointLight.shadow.camera.far = 5


//for visualize the size of the shadow camera use the camera helper
const pointLightCameraHelper = new THREE.CameraHelper(pointLight.shadow.camera)
pointLightCameraHelper.visible = false
scene.add(pointLightCameraHelper)





//BAKING SHADOWS
//Shadows require a lot of work, and the quality is not so good, so if we need only one or few shadow it's okey, but if you have multiple shadow and want a better result, you have to look for other solutions
//one of this is baking shadow , or "blend" it directly within the textures
//the problem is , if you move you object in the scene, the shadow don't move and won't follow the object movement

//NOTE If you have open this project now for the first time, and will see this technique uncomment the code in the material of the plane




//BAKING SHADOW DINAMICALY
// a solution to fix this problem is creating a bake shadow that can move..let's see it
//the shadow move with the sphere with left and right, if you move the sphere up and down it will increase (up) or decrease(down) the alpha of the shadow


//There are other tecniques as well o handle shadow and it's up to you. it depend on the project, what you want do, how many shadow to render and other factor
//and you can also use a combination of this tecnique to find the best solution for your project


//MATERIAL
const material = new THREE.MeshStandardMaterial()
material.metalness = 0.0
material.roughness = 0.7
gui.add(material, "roughness").min(0).max(1).step(0.001)

// SPHERE
const sphereGeometry = new THREE.SphereGeometry()
const sphere = new THREE.Mesh(sphereGeometry, material)
sphere.castShadow = true
sphere.receiveShadow = true
scene.add(sphere)

//PLANE
const planeGeometry = new THREE.PlaneBufferGeometry(10,10)
const plane = new THREE.Mesh(planeGeometry, material  /* , new THREE.MeshBasicMaterial({map: bakeShadow})*/)
plane.position.y -= 1
plane.rotation.x -= Math.PI /2
plane.receiveShadow = true
scene.add(plane)

const sphereShadow = new THREE.Mesh(
  new THREE.PlaneBufferGeometry(2,2),
  new THREE.MeshBasicMaterial({color:0x000000, transparent:true, alphaMap: simpleShadow})
)
sphereShadow.rotation.x = - Math.PI * 0.5
sphereShadow.position.y = plane.position.y + 0.01
scene.add(sphereShadow)



// CAMERA
let camera = new THREE.PerspectiveCamera(
  75, 
  sizes.width / sizes.height, 
  0.1,
  100  
);
camera.position.set(0,0,3)
scene.add(camera);

const clock = new THREE.Clock()

//Animate Loop
const animate = () =>{
  const elapsedTime = clock.getElapsedTime()

  //update the sphere to see the shadow moving
  sphere.position.x = Math.cos(elapsedTime) * 1.5
  sphere.position.z = Math.sin(elapsedTime) * 1.5
  sphere.position.y = Math.abs(Math.sin(elapsedTime) * 3)

  //update the shadow
  sphereShadow.position.x = sphere.position.x
  sphereShadow.position.z = sphere.position.z
  sphereShadow.material.opacity = 1 - sphere.position.y * 0.3

  controls.update()
  
  renderer.render(scene, camera)
}

onMounted(() => {
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
  });
 
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2)) // in questo modo il max valore di pixel ratio utilizzaro è 2 (altrimenti devi renderizzare troppi pixel, dispiego enorme di potenza gpu)
  renderer.shadowMap.enabled = false
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.render(scene, camera)
  renderer.setAnimationLoop(animate)

  //CONTROLS
  controls = new OrbitControls(camera, canvasRef.value)
  controls.enableDamping = true; // permette uno effetto smooth una volta che rilasciamo l'elemento, rallenta fino a fermarsi
});

onUnmounted(() => {
  renderer.setAnimationLoop(null)
});

</script>

<style>
canvas {
  width: 100%;
  height: 100%;
  display: block;
}
</style>