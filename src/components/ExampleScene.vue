<script setup lang="ts">
import { useGLTF } from "@tresjs/cientos";
import { useLoop } from "@tresjs/core";
import {ref} from "vue";

const donutRef = ref();

const { onBeforeRender } = useLoop();

const modelPath = '/models/exampleKeyboard.glb'
const { state: model } = useGLTF(modelPath)

onBeforeRender(({ elapsed }) => {
    if (donutRef.value) {
        donutRef.value.rotation.x = elapsed * .5
        donutRef.value.rotation.y = elapsed * .5
    }
})

</script>

<template>
    <TresPerspectiveCamera
        :position="[7 ,7, 7]"
        :look-at="[0, 0, 0]"
    />

    <TresAmbientLight/>
    <TresDirectionalLight :position="[0, 2, 4]"/>

    <!-- <TresAxesHelper/> -->
    <TresGridHelper/>

     <primitive v-if="model" :object="model.scene"/>

</template>
