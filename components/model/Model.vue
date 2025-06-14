<template>
  <div class="">
    <div class="flex flex-col justify-center items-center">
      <div
        ref="baseDomObject"
        :class="mdAndUp ? 'baseDom-md' : 'baseDom-sm'"
        @dblclick="onHalfHeartPressed"
      />
      <div
        ref="threeDControls"
        class="baseModelControl"
        :class="mdAndUp ? 'baseModelControl-md' : 'baseModelControl-sm'"
      >
        <div class="baseModelCB" :class="mdAndUp ? 'baseModelCB-md' : ''">
          <button
            class="absolute top-0 left-0 w-1/5 h-full hover:bg-gray-50/10"
            @click="onResetAllModelsView"
          />
          <img
            src="~/assets/images/gestures-icons.png"
            class="h-full w-full md:object-contain"
          />
          <button
            class="absolute top-0 right-0 w-1/4 h-full hover:bg-gray-50/10"
            @click="onHalfHeartPressed"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      heartRate: 2500,
      threeDControlsHeight: 0,
      Copper: null,
      THREE: null,
      baseRenderer: null,
      oldCam: null,
      scene: null,
      modelToSceneArray: [],
      container: null,
      isModelHalfed: false,
      modelName: "",
      modelURLsArray: {
        NoInfarct_highres: [
          "heartInfarct/noInfarct.glb",
          "modelView/noInfarct_view_setmodel.json",
        ],
        NormalElectricity_highres: [
          "heartElectricity/normalActivity.glb",
          "modelView/noInfarct_view_setmodel.json",
        ],
        CompensatedFailure_highres: [
          "heartFailure/compensated.glb",
          "modelView/noInfarct_view_setmodel.json",
        ],
        SmallInfarct_highres: [
          "heartInfarct/smallInfarct.glb",
          "modelView/noInfarct_view_setmodel.json",
        ],
        LargeInfarct_highres: [
          "heartInfarct/largeInfarct.glb",
          "modelView/noInfarct_view_setmodel.json",
        ],
        ArrythmiaElectricity_highres: [
          "heartElectricity/arrythmiaActivity.glb",
          "modelView/arrythmiaActivity_view.json",
        ],
        DecompensatedFailure_highres: [
          "heartFailure/decompensated.glb",
          "modelView/noInfarct_view_setmodel.json",
        ],
      },
    };
  },

  props: {
    totalHeight: {
      type: Number,
    },
    availableHeight: {
      type: Number,
    },
  },

  computed: {
    mdAndUp() {
      return this.$vuetify.breakpoint.mdAndUp;
    },
  },

  mounted() {
    this.THREE = this.$three();
    this.modelName = this.$model().name;
    this.container = this.$refs.baseDomObject;
    const baseContainer = this.$baseContainer();
    this.isModelHalfed = this.$isHalfModel();
    setTimeout(() => {
      this.mdAndUp
        ? (baseContainer.style.height = "100vh")
        : (baseContainer.style.height = "400px");
    }, 100);
    this.container.appendChild(baseContainer);
    this.start();

    window.addEventListener(
      "touchstart",
      (event) => {
        event.preventDefault();
        event.stopPropagation();
        var len = event.touches.length;
        if (len === 3) {
          this.scene.controls.noZoom = true;
          this.scene.controls.noRotate = true;
          this.scene.controls.staticMoving = false;
        }
      },
      false
    );
    window.addEventListener(
      "touchend",
      () => {
        this.scene.controls.staticMoving = true;
        setTimeout(() => {
          this.scene.controls.noZoom = false;
          this.scene.controls.noRotate = false;
        }, 100);
      },
      false
    );
  },

  methods: {
    start() {
      this.Copper = this.$Copper();
      this.baseRenderer = this.$baseRenderer();
      this.modelToSceneArray = this.$modelToSceneArray();

      if (
        // this.modelName === "NoInfarct" ||
        this.modelName === "SmallInfarct" ||
        this.modelName === "LargeInfarct" ||
        this.modelName === "CompensatedFailure" ||
        this.modelName === "DecompensatedFailure" ||
        this.modelName === "NormalElectricity"
      ) {
        this.oldCam = this.$perviousCamera();
      }
      if (this.$model().name === "ArrythmiaElectricity") {
        this.loadModel(this.modelName, 0.25);
      } else {
        this.loadModel(this.modelName, 1.0);
      }

      this.heartRate = this.$heartBeat();
      this.updateSlider(this.heartRate);
    },
    loadModel(model_name, rateScaling) {
      let model_prefix = "_highres";
      let metaURL = "";
      let viewURL = "";
      const urls = [];

      if (model_name !== "NoInfarct") {
        metaURL = this.modelURLsArray[model_name + model_prefix][0];
        viewURL = this.modelURLsArray[model_name + model_prefix][1];
      } else {
        for (let i = 1; i <= 1; i++) {
          urls.push(`dynamicImage/one_frame/${i}.dcm`);
        }
        // metaURL = "dynamicImage/heart_model_grey.gltf";
        metaURL = "dynamicImage/one_frame/heart_p.gltf";
        viewURL = "dynamicImage/texture2d_view_array.json";
      }

      this.scene = this.baseRenderer.getSceneByName(model_name);
      if (this.scene === undefined) {
        this.scene = this.baseRenderer.createScene(model_name);
        this.scene.controls.staticMoving = true;
        // this.scene.controls.rotateSpeed = 2.0;
        this.baseRenderer.setCurrentScene(this.scene);
        if (model_name !== "NoInfarct") {
          this.scene.loadGltf(metaURL, (content) => {
            if (model_name === "ArrythmiaElectricity") {
              this.scene.setModelPosition(content, { x: 5, y: 2 });
            } else {
              this.scene.setModelPosition(content, { y: 3 });
            }

            if (this.oldCam && this.oldCam.near) {
              this.shareCameraSettings(this.oldCam);
            }
          });
        } else {
          //   "gltfloader-plugin-test": "^1.8.16",
          this.scene.loadDicom(urls, {
            getCopperVolume(copperVolume, updateTexture) {
              copperVolume.windowWidth = 424;
              copperVolume.windowCenter = 236;
              updateTexture(copperVolume);
            },
            setAnimation(currentValue, depth, depthStep) {
              currentValue += depthStep;
              if (currentValue > depth) {
                currentValue = 0;
              }
              return currentValue;
            },
          });
          this.scene.loadGltf(metaURL, (content) => {
            content.rotation.set(-12.9, 3.6, 3.1);
            content.scale.set(3.4, 3.4, 3.4);
            content.position.set(4.3, 1.3, 0);

            this.scene?.setPlayRate(3.5);
          });
        }

        this.$store.commit("setModelToSceneArray", this.scene);

        this.scene.loadViewUrl(viewURL);
        this.scene.updateBackground("#000", "#000");
        this.Copper.setHDRFilePath("environment/footprint_court_2k.hdr");
        this.baseRenderer.updateEnvironment();
        this.addLabel(this.modelName);
      } else {
        this.meshReady(this.oldCam);
        this.baseRenderer.setCurrentScene(this.scene);
      }
    },
    meshReady(oldCam) {
      if (this.oldCam && oldCam.near) {
        this.shareCameraSettings(oldCam);
      }
    },

    shareCameraSettings(oldCam) {
      if (
        oldCam !== null &&
        oldCam.near !== null &&
        oldCam.near !== undefined
      ) {
        const target = [
          -0.9551143646240234, 2.91867446899414, 2.7563438415527344,
        ];
        const viewpoint = this.scene.setViewPoint(oldCam, target);
        this.scene.updateCamera(viewpoint);
        if (this.isModelHalfed != this.scene.isHalfed) this.showHalf();
        this.scene.isHalfed = this.isModelHalfed;
      }
    },

    onResetAllModelsView() {
      this.heartRate = 2500;
      this.isModelHalfed = false;
      $nuxt.$emit("beat-reset", 2500);
      this.$store.commit("setHeartBeat", 2500);
      this.$store.commit("setPreviousCamera", {});
      this.$store.commit("setIsHalfModel", false);

      for (var k in this.modelToSceneArray) {
        if (this.modelToSceneArray.hasOwnProperty(k)) {
          this.modelToSceneArray[k].resetView();
          if (this.modelToSceneArray[k].isHalfed) {
            this.showHalf(this.modelToSceneArray[k]);
          }
        }
      }
    },
    convertHeartRate(heartRate) {
      if (this.modelName === "ArrythmiaElectricity") {
        return heartRate / 2000;
      } else {
        return heartRate / 500;
      }
    },
    updateSlider(heartRate) {
      const convertRate = this.convertHeartRate(heartRate);
      this.scene.setPlayRate(convertRate);
    },
    addLabel(model_name) {
      // if (model_name === "NoInfarct" || model_name === "NormalElectricity") {
      if (model_name === "NormalElectricity") {
        this.Copper.addLabelToScene(
          this.scene,
          "right ventricle",
          -45.323991175632,
          44.1417335930078,
          10.421283,
          60.0
        );
        this.Copper.addLabelToScene(
          this.scene,
          "left ventricle",
          -55.056679,
          4.82123313284426,
          5.421283,
          60.0
        );
      } else if (model_name === "SmallInfarct") {
        this.Copper.addLabelToScene(
          this.scene,
          "damaged tissue",
          30,
          -40,
          0,
          60.0
        );
      } else if (model_name === "LargeInfarct") {
        this.Copper.addLabelToScene(
          this.scene,
          "damaged tissue",
          15,
          -45,
          0,
          60.0
        );
      }
    },
    onHalfHeartPressed() {
      this.isModelHalfed = !this.isModelHalfed;
      this.showHalf();
    },
    showHalf(sceneObj) {
      let scene;
      if (sceneObj) {
        scene = sceneObj;
      } else {
        scene = this.baseRenderer.getSceneByName(this.modelName);
      }
      this.$store.commit("setIsHalfModel", this.isModelHalfed);

      if (this.modelName === "NormalElectricity") {
        scene.content.traverse((child) => {
          if (child.name === "Ant") {
            scene.updateModelChildrenVisualisation(child);
          }
        });
      } else {
        scene.content.traverse((child) => {
          if (
            child.name === "Post_top" ||
            child.name === "Post_inner" ||
            child.name === "Post_NonInfarct" ||
            child.name === "Post_top_1" ||
            child.name === "Post_inner_1" ||
            child.name === "Post_NonInfarct_1"
          ) {
            scene.updateModelChildrenVisualisation(child);
          }
        });
      }
    },
  },

  watch: {
    heartRate: function (currentRate) {
      this.updateSlider(currentRate);
    },
  },

  created() {
    this.$nuxt.$on("beat-update-onTime", (immediateBeat) => {
      this.updateSlider(immediateBeat);
    });
    this.$nuxt.$on("beat-change", (currentBeat) => {
      this.heartRate = currentBeat;
      this.$store.commit("setHeartBeat", currentBeat);
    });
  },

  beforeDestroy() {
    if (this.oldCam) {
      const currentCamera = this.baseRenderer.getCurrentScene().camera;
      const position = new this.THREE.Vector3();
      const up = new this.THREE.Vector3();
      position.copy(currentCamera.position);
      up.copy(currentCamera.up);

      const currentCameraInfo = {
        position: position,
        up: up,
        near: currentCamera.near,
        far: currentCamera.far,
      };

      this.$store.commit("setPreviousCamera", currentCameraInfo);
    }

    this.$nuxt.$off("beat-change");
  },
};
</script>

<style scoped lang="scss">
.baseModelControl {
  width: 100vw;
  height: 120px;
  // background: red;
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-content: center;
  .baseModelCB {
    width: 240px;
    height: 70px;
    position: relative;
  }
  .baseModelCB-md {
    width: 280px;
    height: 100px;
  }
}

.baseModelControl-md {
  position: fixed;
  bottom: 10px;
  padding-left: 100px;
}
.baseModelControl-sm {
  order: -1;
  height: 60px;
}
.baseDom-md {
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
}
.baseDom-sm {
  width: 100vw;
  height: 400px;
  margin: 0;
  padding: 0;
}
.outer-model {
  height: 100%;
}
.hidden-input {
  width: 1px;
  height: 1px;
}
</style>
