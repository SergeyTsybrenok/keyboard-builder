<script setup lang="ts">
import { useGLTF } from "@tresjs/cientos";
import { useLoop } from "@tresjs/core";
import { ref } from "vue";

const props = defineProps({
  color: String,
});

const donutRef = ref();
// const color = ref("#7dc4e4");

const { onBeforeRender } = useLoop();

const modelPath = "/models/exampleKeyboard.glb";
const { state: model } = useGLTF(modelPath);

onBeforeRender(({ elapsed }) => {
  if (donutRef.value) {
    donutRef.value.rotation.x = elapsed * 0.5;
    donutRef.value.rotation.y = elapsed * 0.5;
  }
});
</script>

<template>
  <TresPerspectiveCamera :position="[7, 7, 7]" :look-at="[0, 0, 0]" />

  <TresAmbientLight />
  <TresDirectionalLight :position="[0, 2, 4]" />

  <!-- <TresAxesHelper/> -->
  <TresGridHelper />

  <TresMesh>
    <primitive v-if="model" :object="model.scene" :ref="donutRef" />
        <TresMeshStandardMaterial :color="props.color" :metalness="0" />
  </TresMesh>
  <TresMesh :cast-shadow="true" :position="[0, 1, 0]">
    <TresBoxGeometry />
    <!-- <TresMeshBasicMaterial :color="props.color" /> -->
        <TresMeshStandardMaterial :color="props.color" :metalness="0" />
  </TresMesh>
</template>
