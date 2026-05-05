<script setup lang="ts">
import { TresCanvas } from "@tresjs/core";
import ExampleScene from "./components/ExampleScene.vue";
import { OrbitControls } from "@tresjs/cientos";
import { ref } from "vue";
import { LinearToneMapping } from "three";

const color = ref("#7dc4e4");
</script>

<template>
  <div class="container-fluid">
    <div class="row g-0">
      <div
        class="col-md-6"
        style="
          position: absolute;
          z-index: 1;
          color: white;
          background-color: #181926;
        "
      >
        <h1 class="display-4">Keyboard Builder</h1>
        <h2 class="display-8">Pre-alpha</h2>

        <button
          class="btn btn-primary"
          @click="color = '#91d7e3'"
          style="background-color: #91d7e3"
        >
          Change color 1
        </button>
        <button class="btn btn-secondary" @click="color = '#eed49f'">
          Change color 2
        </button>
        <button class="btn btn-primary" @click="color = '#c6a0f6'">
          Change color 3
        </button>
      </div>
      <div class="col-md-6">
        <!-- TODO :fps-limit="60"-->
        <TresCanvas
          shadows
          clear-color="#24273a"
          window-size
        >
          <ExampleScene :color="color" />
          <OrbitControls />
        </TresCanvas>
      </div>
    </div>
  </div>
</template>
