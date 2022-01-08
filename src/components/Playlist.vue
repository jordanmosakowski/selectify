<template>
    <div v-on:click='onClick' class='main'>
        <div class='container'>
            <div class='dummy'/>
            <img :src='data.images[0].url'>
        </div>
        <h3> {{data.name}}</h3>
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
    .main{
        cursor: pointer;
        transition: all 0.2s ease-in-out;
        border-radius: 10%;
    }
    .main:hover{
        transform: scale(1.1);
        background-color: #222222;
        box-shadow: rgba(0, 0, 0, 0.25) 0px 54px 55px, rgba(0, 0, 0, 0.12) 0px -12px 30px, rgba(0, 0, 0, 0.12) 0px 4px 6px, rgba(0, 0, 0, 0.17) 0px 12px 13px, rgba(0, 0, 0, 0.09) 0px -3px 5px;
    }
    .container {
        display: inline-block;
        position: relative;
        width: 100%;
    }
    .dummy {
        margin-top: 100%;
    }
    img {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        border-radius: 10%;
        object-fit: cover;
    }
</style>