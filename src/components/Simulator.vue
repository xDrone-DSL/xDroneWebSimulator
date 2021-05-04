<template>
  <div>
    <div id="container"></div>
    <v-row style="width: 800px">
      <v-col :cols="3">
        <v-btn class="ma-2 pa-2" @click="reloadPage">Replay</v-btn>
      </v-col>
      <v-col>
        <v-row dense>
          <v-col :cols="2">
            <span>Speed:</span>
          </v-col>
          <v-col>
            <v-radio-group class="ma-0 pa-0" v-model="playSpeed" row mandatory hide-details>
              <v-radio label="Pause" :value="0"></v-radio>
              <v-radio label="0.5x" :value="0.5"></v-radio>
              <v-radio label="1x" :value="1"></v-radio>
              <v-radio label="2x" :value="2"></v-radio>
            </v-radio-group>
          </v-col>
        </v-row>
        <v-row dense>
          <v-col :cols="2">
            FPS
            <v-tooltip bottom>
              <template v-slot:activator="{ on, attrs }">
                <v-icon
                    v-bind="attrs"
                    v-on="on"
                >
                  mdi-help-box
                </v-icon>
              </template>
              <div>A higher FPS improves the accuracy, but causes a heavier burden on you browser</div>
            </v-tooltip>
            :
          </v-col>
          <v-col>
            <v-radio-group class="ma-0 pa-0" v-model="framesPerSecond" row mandatory hide-details>
              <v-radio label="30fps" :value="30"></v-radio>
              <v-radio label="60fps" :value="60"></v-radio>
              <v-radio label="90fps" :value="90"></v-radio>
            </v-radio-group>
          </v-col>
        </v-row>
      </v-col>
    </v-row>
    <div class="ml-1">
      <div>Rotate Camera: Click and Drag</div>
      <div>Move Camera: Press Ctrl + Click and Drag</div>
      <div>Zoom In/Out: Scroll</div>
    </div>
  </div>
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
      DISTANCE_SCALE_FACTOR: 20,
      framesPerSecond: 30,
      playSpeed: 1,
      groups: new Map(),
      renderer: null,
      scene: null,
      camera: null,
      controls: null,
      mixers: new Map(),
      animTime: 0,
      axis10cm: null,
      axis1m: null,
      start: true,
      play: true
    };
  },
  methods: {
    moveX: function (group, meter) {
      group.translateX(-meter * this.DISTANCE_SCALE_FACTOR);
    },
    moveY: function (group, meter) {
      group.translateZ(meter * this.DISTANCE_SCALE_FACTOR);
    },
    moveZ: function (group, meter) {
      group.translateY(meter * this.DISTANCE_SCALE_FACTOR);
    },
    rotate: function (group, degree) {
      group.rotation.y -= (degree / 180 * Math.PI);
    },
    reloadPage() {
      window.location.reload();
    },
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
          this.moveX(this.groups.get(name), init_pos[0]);
          this.moveY(this.groups.get(name), init_pos[1]);
          this.moveZ(this.groups.get(name), init_pos[2]);
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
      this.camera.position.y = 40;
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
    getSpeed: function (name) {
      return this.config.find(c => c.name == name).speed;
    },
    getRotateSpeed: function (name) {
      return this.config.find(c => c.name == name).rotate_speed;
    },
    getTakeoffHeight: function (name) {
      return this.config.find(c => c.name == name).takeoff_height;
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
            currCommand.values = [this.getTakeoffHeight(name)];
            break;
          case "land":
            currCommand.action = "down";
            currCommand.values = [group.position.y / this.DISTANCE_SCALE_FACTOR];
            break;
          case "forward":
            this.moveY(group, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "backward":
            this.moveY(group, -this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "right":
            this.moveX(group, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "left":
            this.moveX(group, -this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "up":
            this.moveZ(group, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "down":
            this.moveZ(group, -this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "rotate_right":
            this.rotate(group, this.getRotateSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getRotateSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "rotate_left":
            this.rotate(group, -this.getRotateSpeed(name) / this.framesPerSecond * this.playSpeed);
            this.updateSequential(sequential, this.getRotateSpeed(name) / this.framesPerSecond * this.playSpeed);
            break;
          case "wait":
            this.updateSequential(sequential, 1 / this.framesPerSecond * this.playSpeed);
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
