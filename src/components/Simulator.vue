<template>
  <div id="container"></div>
</template>

<script>
import * as THREE from "three";
import {OrbitControls} from "three/examples/jsm/controls/OrbitControls.js";
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader.js";
import {Sky} from "three/examples/jsm/objects/Sky.js";
import particleFire from "three-particle-fire";

particleFire.install({THREE: THREE});

export default {
  props: {
    commands: {type: Array, required: true},
    config: {type: Array, required: true}
  },
  data() {
    return {
      groups: new Map(),
      renderer: null,
      scene: null,
      camera: null,
      controls: null,
      mixers: new Map(),
      animTime: 0,
      framesPerSecond: 30,
      axis10cm: null,
      axis1m: null,
      start: true,
      play: true
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
    init: function () {
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
          new THREE.MeshBasicMaterial({color: 0xffffff})
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
      this.renderer = new THREE.WebGLRenderer({antialias: true, alpha: true});
      this.renderer.setSize(800, 600);
      container.appendChild(this.renderer.domElement);

      let loader = new GLTFLoader();
      // Load Drone
      this.config.forEach((config) => {
        const name = config.name;
        const init_pos = config.init_pos;
        loader.load("/models/drone/scene.gltf", obj => {
          this.groups.set(name, new THREE.Group());

          // remove bad shadow
          obj.scene.children[0].children[0].children[0].children.pop(12);
          obj.scene.position.y += 1.5;
          this.groups.get(name).add(obj.scene);
          this.groups.get(name).translateX(-init_pos[0]);//TODO: scale
          this.groups.get(name).translateZ(init_pos[1]);//TODO: scale
          this.groups.get(name).translateY(init_pos[2]);//TODO: scale
          this.scene.add(this.groups.get(name));
          this.mixers.set(name, new THREE.AnimationMixer(this.groups.get(name)));
          obj.animations.forEach(clip => {
            this.mixers.get(name).clipAction(clip).play();
          });
        });

      })
      // Ground
      const textureLoader = new THREE.TextureLoader();
      const groundColor = textureLoader.load("/models/grass_diffuse.png");
      groundColor.repeat.set(100, 100);
      groundColor.wrapS = groundColor.wrapT = THREE.RepeatWrapping;

      const ground = new THREE.Mesh(
          new THREE.PlaneBufferGeometry(2000, 2000, 1, 1),
          new THREE.MeshPhongMaterial({map: groundColor})
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
    animate: function () {
      setTimeout(() => {
        requestAnimationFrame(this.animate);
      }, 1000 / this.framesPerSecond);

      //crop animation
      if (this.play && this.mixers.size == this.config.length) {
        if (this.animTime < 2.1 || this.animTime >= 3.9) {
          this.mixers.forEach((mixer) => {
            mixer.setTime(2.1);
          })
          this.animTime = 2.1;
        } else {
          this.mixers.forEach((mixer) => {
            mixer.update(0.05);
          })
          this.animTime += 0.05;
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
        this.move(this.commands);
      }
      this.controls.update();

      this.renderer.render(this.scene, this.camera);
    },
    updateSequential: function (sequential, stepSize) {
      sequential[0].values[0] -= stepSize;
      if (sequential[0].values[0] <= 0) {
        sequential.shift();
      }
    },
    move: function (sequential) {
      if (sequential.length == 0) {
        if (this.commands.length == 0) {
          this.play = false;
        }
        return;
      }
      const currCommand = sequential[0];
      if (currCommand.type == "single") {
        const name = currCommand.name;
        const group = this.groups.get(name);
        switch (currCommand.action) {
          case "takeoff":
            currCommand.action = "up";
            currCommand.values = [this.config.find(c => c.name == name).takeoff_height];
            break;
          case "land":
            currCommand.action = "down";
            currCommand.values = [group.position.y / 3.2]; //TODO: what magic number is this????
            break;
          case "forward":
            group.translateZ(this.speed);//todo change speed
            this.updateSequential(sequential, 2 / this.framesPerSecond);
            break;
          case "backward":
            group.translateZ(-this.speed);
            this.updateSequential(sequential, 2 / this.framesPerSecond);
            break;
          case "right":
            group.translateX(-this.speed);
            this.updateSequential(sequential, 2 / this.framesPerSecond);
            break;
          case "left":
            group.translateX(this.speed);
            this.updateSequential(sequential, 2 / this.framesPerSecond);
            break;
          case "up":
            group.translateY(this.speed);
            this.updateSequential(sequential, 5 / this.framesPerSecond);
            break;
          case "down":
            group.translateY(-this.speed);
            this.updateSequential(sequential, 5 / this.framesPerSecond);
            break;
          case "rotate_right":
            group.rotation.y -= this.angleSpeedRadius;
            this.updateSequential(sequential, this.angleSpeedDegree);
            break;
          case "rotate_left":
            group.rotation.y += this.angleSpeedRadius;
            this.updateSequential(sequential, this.angleSpeedDegree);
            break;
          case "wait":
            this.updateSequential(sequential, 1 / this.framesPerSecond);
            break;
        }

      }
      if (currCommand.type == "parallel") {
        currCommand.branches.forEach((branch) => {
          this.move(branch)
        })
        if (currCommand.branches.every((branch) => branch.length == 0)) {
          sequential.shift();
        }
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
