<template>
  <div id="container"></div>
</template>

<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { Sky } from "three/examples/jsm/objects/Sky.js";
import particleFire from "three-particle-fire";
particleFire.install({ THREE: THREE });

export default {
  props: {
    animation: { type: Array, required: true },
    environments: { required: true }
  },
  data() {
    return {
      group: null,
      renderer: null,
      scene: null,
      camera: null,
      controls: null,
      mixer: null,
      animTime: 0,
      framesPerSecond: 30,
      axis10cm: null,
      axis1m: null,
      start: true,
      play: true,
      fires: []
    };
  },
  computed: {
    angleSpeedRadius() {
      return Math.PI / 2 / this.framesPerSecond;
    },
    angleSpeedDegree() {
      return 90 / this.framesPerSecond;
    },
    speed() {
      return 4 / 3 / 2.5;
    }
  },
  methods: {
    init: function() {
      let container = document.getElementById("container");
      this.camera = new THREE.PerspectiveCamera(70, 8 / 6, 1, 1000);

      this.scene = new THREE.Scene();

      // Add Sky
      const sky = new Sky();
      sky.scale.setScalar(45000);
      this.scene.add(sky);

      // Add Sun Helper
      const sunSphere = new THREE.Mesh(
        new THREE.SphereBufferGeometry(20000, 16, 8),
        new THREE.MeshBasicMaterial({ color: 0xffffff })
      );
      sunSphere.position.y = -700000;
      sunSphere.visible = false;
      this.scene.add(sunSphere);

      const distance = 400000;

      sky.material.uniforms["turbidity"].value = 10;
      sky.material.uniforms["rayleigh"].value = 2;
      sky.material.uniforms["mieCoefficient"].value = 0.005;
      sky.material.uniforms["mieDirectionalG"].value = 0.8;
      sky.material.uniforms["luminance"].value = 1;

      // inclination
      const theta = Math.PI * (0.2364 - 0.5);
      // azimuth
      const phi = 2 * Math.PI * (0.25 - 0.5);
      sunSphere.position.x = distance * Math.cos(phi);
      sunSphere.position.y = distance * Math.sin(phi) * Math.sin(theta);
      sunSphere.position.z = distance * Math.sin(phi) * Math.cos(theta);
      sunSphere.visible = true;
      sky.material.uniforms["sunPosition"].value.copy(sunSphere.position);

      let light = new THREE.AmbientLight("#ffffff"); // soft white light
      this.scene.add(light);

      //Helper
      this.axis10cm = new THREE.GridHelper(2000, 1000);
      this.axis10cm.position.y = 0;
      this.axis10cm.material.opacity = 0.5;
      this.axis10cm.material.transparent = true;
      this.scene.add(this.axis10cm);

      this.axis1m = new THREE.GridHelper(2000, 100);
      this.axis1m.position.y = 0;
      this.axis1m.material.opacity = 1;
      this.axis1m.material.transparent = true;
      this.scene.add(this.axis1m);

      // Renderer
      this.renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      this.renderer.setSize(800, 600);
      container.appendChild(this.renderer.domElement);

      let loader = new GLTFLoader();
      // Load Drone
      loader.load("/models/drone/scene.gltf", obj => {
        this.group = new THREE.Group();

        // remove bad shadow
        obj.scene.children[0].children[0].children[0].children.pop(12);
        obj.scene.position.y += 1.5;
        this.group.add(obj.scene);
        this.scene.add(this.group);
        this.mixer = new THREE.AnimationMixer(this.group);
        obj.animations.forEach(clip => {
          this.mixer.clipAction(clip).play();
        });
      });

      // Ground
      const textureLoader = new THREE.TextureLoader();
      const groundColor = textureLoader.load("/models/grass_diffuse.png");
      groundColor.repeat.set(100, 100);
      groundColor.wrapS = groundColor.wrapT = THREE.RepeatWrapping;

      const ground = new THREE.Mesh(
        new THREE.PlaneBufferGeometry(2000, 2000, 1, 1),
        new THREE.MeshPhongMaterial({ map: groundColor })
      );
      ground.position.y = -0.3;
      ground.rotation.x = THREE.Math.degToRad(-90);
      this.scene.add(ground);

      // Controls
      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.camera.position.x = 10;
      this.camera.position.y = 24;
      this.camera.position.z = 40;
      this.controls.maxPolarAngle = Math.PI / 2;
      this.controls.autoRotate = true;
      this.controls.autoRotateSpeed = -25;
      this.controls.dampingFactor = true;
      this.camera.zoom = 8;
      this.camera.updateProjectionMatrix();

      this.controls.update();
    },
    animate: function() {
      setTimeout(() => {
        requestAnimationFrame(this.animate);
        this.fires.forEach(f => {
          f.material.update(0.05);
        });
      }, 1000 / this.framesPerSecond);
      //crop animation
      if (this.play && this.mixer !== null) {
        if (this.animTime < 2.1 || this.animTime >= 3.9) {
          this.mixer.setTime(2.1);
          this.animTime = 2.1;
        } else {
          this.animTime += 0.05;
          this.mixer.update(0.05);
        }
      }

      if (this.start) {
        this.camera.zoom *= 0.97;
        this.camera.updateProjectionMatrix();
        if (this.camera.zoom < 1.24) {
          this.controls.autoRotate = false;
          this.start = false;
        }
      } else {
        this.move();
      }
      this.controls.update();

      this.renderer.render(this.scene, this.camera);
    },
    updateAndCheckFinished: function(stepSize) {
      this.animation[0].value[0] -= stepSize;
      if (this.animation[0].value[0] <= 0) {
        this.animation.shift();
        return true;
      }
      return false;
    },
    move: function() {
      if (this.group.position.y < 0 || this.animation.length == 0) {
        this.play = false;
        return;
      }
      switch (this.animation[0].action) {
        case "takeoff":
          this.animation[0] = { action: "up", value: [2] };
          break;
        case "land":
          this.animation[0] = {
            action: "down",
            value: [this.group.position.y]
          };
          break;
        case "forward":
          this.group.translateZ(this.speed);
          if (this.updateAndCheckFinished(2 / this.framesPerSecond)) return;
          break;
        case "backward":
          this.group.translateZ(-this.speed);
          if (this.updateAndCheckFinished(2 / this.framesPerSecond)) return;
          break;
        case "right":
          this.group.translateX(-this.speed);
          if (this.updateAndCheckFinished(2 / this.framesPerSecond)) return;
          break;
        case "left":
          this.group.translateX(this.speed);
          if (this.updateAndCheckFinished(2 / this.framesPerSecond)) return;
          break;
        case "up":
          this.group.translateY(this.speed);
          if (this.updateAndCheckFinished(5 / this.framesPerSecond)) return;
          break;
        case "down":
          this.group.translateY(-this.speed);
          if (this.updateAndCheckFinished(5 / this.framesPerSecond)) return;
          break;
        case "rotate_right":
          this.group.rotation.y -= this.angleSpeedRadius;
          if (this.updateAndCheckFinished(this.angleSpeedDegree)) return;
          break;
        case "rotate_left":
          this.group.rotation.y += this.angleSpeedRadius;
          if (this.updateAndCheckFinished(this.angleSpeedDegree)) return;
          break;
        case "wait":
          if (this.updateAndCheckFinished(1 / this.framesPerSecond)) return;
          break;
      }
    }
  },
  mounted() {
    this.init();
    this.animate();
  }
};
</script>

<style>
#container {
  width: 800px;
  height: 600px;
}
</style>
