<script setup lang="ts">
import { TresCanvas } from "@tresjs/core";
import ExampleScene from "./components/ExampleScene.vue";
import { OrbitControls } from "@tresjs/cientos";
</script>

<template>
  <div class="container-fluid">
    <div class="row g-0">
      <div class="col-md-6" style="position: absolute; z-index: 1; color: white; background-color: #181926;">
        <h1 class="display-4">Keyboard Builder</h1>
        <h2 class="display-8">Pre-alpha</h2>
        <button class="btn btn-primary">Configurate</button>
      </div>
      <!-- <div class="col-md-6"> -->
        <TresCanvas shadows clear-color="#24273a" window-size>
          <ExampleScene />
          <OrbitControls />
        </TresCanvas>
      </div>
    </div>
  <!-- </div> -->
</template>