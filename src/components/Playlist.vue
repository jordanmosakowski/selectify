<template>
    <div>
        <img :src='data.images[0].url'>
        <h3> {{data.name}}</h3>
    </div>
</template>

<script>
export default {
    name: "Playlist",
    props: {
        data: Object
    },
}
</script>

<style scoped>
    img{
        width: 100%;
        border-radius: 5%;
    }
</style>