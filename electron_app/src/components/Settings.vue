<template>
  <div class="animatable_content_box">
    <div v-if="!is_custom_model_loading">
      <h1>설정</h1>
      <br />
      <h2>일반</h2>
      <br />
      <div class="setting_box">
        <div class="settings_left">
          <h3>알림 소리</h3>
          <p>이미지 생성이 끝나면 소리로 알려주기.</p>
        </div>
        <hr />
        <div style="float: right; margin-right: 9px; align-self: center">
          <label class="switch">
            <input
              type="checkbox"
              v-model="app_state.app_data.settings.notification_sound"
              checked
            />
            <span class="toggle round"></span>
          </label>
        </div>
      </div>
      <hr />
      <!-- 
            <div class="setting_box">
            <div class="settings_left">
            <h2>Allow NSFW Content</h2>
            <p>Allows image generation considered NSFW</p>
            </div>
            <hr>
            <div style="float:right;margin-right: 9px;align-self: center;" >
                <label class="switch">
            <input type="checkbox" v-model="app_state.app_data.settings.nsfw_filter">
            <span class="toggle round"></span>
            </label>
            </div>
            </div>
            <hr> -->
      <div
        class="l_button button_colored"
        style="float: right"
        @click="add_model"
      >
        모델 추가
      </div>
      <h2>커스텀 모델</h2>

      <!-- <br> -->
      <hr />
      <div
        v-for="model in Object.values(this.app_state.app_data.custom_models)"
        :key="model.name"
      >
        <div
          class="l_button"
          @click="delete_model(model.name)"
          style="float: right"
        >
          삭제
        </div>
        <p>이름 : {{ model.name }}</p>
        <p>경로 : {{ model.orig_path }}</p>
        <hr />
      </div>
    </div>
    <div v-else>
      <LoaderModal
        :loading_percentage="-1"
        loading_title="Importing model"
      ></LoaderModal>
    </div>
  </div>
</template>
<script>
import LoaderModal from "../components_bare/LoaderModal.vue";
import Vue from "vue";

export default {
  name: "Settings",
  props: {
    app_state: Object,
  },
  components: { LoaderModal },
  mounted() {},
  data() {
    return {
      is_custom_model_loading: false,
    };
  },
  methods: {
    delete_model(k) {
      let model_path = this.app_state.app_data.custom_models[k].path;
      window.ipcRenderer.sendSync("delete_file", model_path);
      Vue.delete(this.app_state.app_data.custom_models, k);
    },
    add_model() {
      let that = this;

      let pytorch_model_path = window.ipcRenderer.sendSync(
        "file_dialog",
        "ckpt_file"
      );
      if (!pytorch_model_path) return;

      if (pytorch_model_path == "NULL") return;

      let model_name = pytorch_model_path.replaceAll("\\", "/").split("/");
      model_name = model_name[model_name.length - 1].split(".")[0];

      if (model_name.trim() == "") {
        alert("Put non empty model name");
        return;
      }

      if (that.app_state.app_data.custom_models[model_name]) {
        alert("this model name " + model_name + " already exists");
        return;
      }

      that.is_custom_model_loading = true;
      window.ipcRenderer
        .invoke("add_custom_pytorch_models", pytorch_model_path, model_name)
        .then((result) => {
          that.is_custom_model_loading = false;
          if (result.success) {
            Vue.set(that.app_state.app_data.custom_models, model_name, {
              name: model_name,
              path: result.model_path,
              orig_path: pytorch_model_path,
            });
          } else {
            alert("Error " + result.error);
          }
        });
    },
  },
};
</script>
<style></style>
<style scoped>
.switch {
  position: relative;
  display: inline-block;
  width: 60px;
  height: 30px;
}

.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.toggle {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #ccc;
  -webkit-transition: 0.4s;
  transition: 0.4s;
}

.toggle:before {
  position: absolute;
  content: "";
  height: 22px;
  width: 23px;
  left: 5px;
  bottom: 4px;
  background-color: white;
  -webkit-transition: 0.4s;
  transition: 0.4s;
}

input:checked + .toggle {
  background-color: #3e7bfa;
}

input:checked + .toggle:before {
  transform: translateX(27px);
}

.toggle.round {
  border-radius: 34px;
}

.toggle.round:before {
  border-radius: 50%;
}

.setting_box {
  display: flex;
  flex-direction: row;
}

.settings_left {
  flex: 1 1 auto;
}
</style>
