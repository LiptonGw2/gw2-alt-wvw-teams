<template>
    <div style="display:inline-block;">
        <button @click="onClick">{{ buttonText }}</button>
        <input type="file" ref="fileInput" v-bind:accept="accept" @change="onFileChanged($event.target.files)"
            style="display: none;">
    </div>
</template>

<script>
export default {
    props: {
        buttonText: {
            type: String,
            default: "Import"
        },
        accept: {
            type: String
        }
    },
    data() {
        return {}
    },

    methods: {
        onClick() {
            if (this.$refs.fileInput) {
                this.$refs.fileInput.click();
            }
        },

        onFileChanged(files) {
            if (!files || !files.length)
                return;

            const reader = new FileReader();

            reader.onload = e => {
                this.$emit("loaded", e.target.result, files[0].name);
            };

            reader.readAsText(files[0]);
        }
    }
}
</script>