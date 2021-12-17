<template>
    <div>
        <img :src='data.images[0].url'>
        <h3> {{data.name}}</h3>
        <button v-on:click='onClick'>Select</button>
    </div>
</template>

<script>
export default {
    name: "Playlist",
    props: {
        data: Object
    },
    methods: {
        onClick(){
            this.$emit("clicked",this.data);
        }
    }
}
</script>

<style scoped>
    img{
        width: 100%;
        border-radius: 5%;
    }
</style>