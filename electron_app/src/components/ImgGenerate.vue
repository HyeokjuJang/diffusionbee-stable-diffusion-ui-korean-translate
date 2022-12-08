@import '../assets/css/theme.css';
<template>
  <div class="animatable_content_box">
    <div v-if="stable_diffusion.is_backend_loaded">
      <div class="textbox_section">
        <textarea
          v-model="prompt"
          placeholder="프롬프트를 입력해주세요."
          style="
            border-radius: 12px 12px 12px 12px;
            width: calc(100%);
            resize: none;
          "
          class="form-control"
          v-bind:class="{ disabled: !stable_diffusion.is_input_avail }"
          :rows="is_negative_prompt_avail ? 2 : 3"
        ></textarea>

        <textarea
          v-if="is_negative_prompt_avail"
          v-model="negative_prompt"
          placeholder="부정 프롬프트를 입력해주세요."
          style="
            border-radius: 12px 12px 12px 12px;
            width: calc(100%);
            resize: none;
            margin-top: 5px;
          "
          class="form-control negative_prompt_tb"
          v-bind:class="{ disabled: !stable_diffusion.is_input_avail }"
          :rows="1"
        ></textarea>

        <div
          v-if="stable_diffusion.is_input_avail"
          class="content_toolbox"
          style="margin-top: 10px; margin-bottom: -10px"
        >
          <div
            class="l_button button_medium button_colored"
            style="float: right"
            @click="generate_from_prompt"
          >
            생성
          </div>

          <!-- <div style="float:right;"  >
                        <div class="l_button" @click="is_adv_options = !is_adv_options">Advanced options</div>
                    </div> -->

          <SDOptionsDropdown
            :options_model_values="this_object"
            :elements_hidden="['inp_img_strength']"
          >
          </SDOptionsDropdown>

          <div style="float: right; margin-top: -5px">
            <b-dropdown
              id="dropdown-form"
              variant="link"
              ref="dropdown"
              toggle-class="text-decoration-none"
              no-caret
            >
              <template #button-content>
                <div class="l_button" style="margin-right: -20px">
                  스타일 목록
                </div>
              </template>

              <b-dropdown-form style="width: 540px">
                <div
                  style="max-height: calc(100vh - 300px); overflow-y: scroll"
                >
                  <div v-for="modifier in modifiers" :key="modifier[0]">
                    <p>{{ modifier[0] }}</p>
                    <div
                      v-for="tag in modifier[1]"
                      @click="add_style(tag)"
                      :key="tag"
                      class="l_button button_small button_colored"
                      style="float: left; margin-bottom: 7px"
                    >
                      {{ tag }}
                    </div>

                    <div
                      style="clear: both; display: table; margin-bottom: 3px"
                    ></div>
                    <hr />
                  </div>

                  <p>Source : cmdr2</p>
                </div>
              </b-dropdown-form>
            </b-dropdown>
          </div>

          <div
            class="l_button button_medium"
            style="float: right; margin-right: -10px; margin-top: -1px"
            @click="open_arthub"
          >
            프롬프트 아이디어 얻기(외부 페이지)
          </div>
        </div>
        <div
          v-else-if="stable_diffusion.generated_by == 'txt2img'"
          class="content_toolbox"
          style="margin-top: 10px; margin-bottom: -10px"
        >
          <div
            v-if="is_stopping"
            class="l_button button_medium button_colored"
            style="float: right"
            @click="stop_generation"
          >
            멈추는 중 ...
          </div>
          <div
            v-else
            class="l_button button_medium button_colored"
            style="float: right"
            @click="stop_generation"
          >
            중단
          </div>
        </div>
      </div>

      <div v-if="generated_images.length == 1">
        <center>
          <ImageItem
            :app_state="app_state"
            :path="generated_images[0]"
            :style_obj="{ width: 'calc(100vh - 390px )', 'margin-top': '60px' }"
          ></ImageItem>
        </center>
      </div>

      <div>
        <br />
        <b-row
          v-if="generated_images.length > 1"
          class="justify-content-md-center"
        >
          <b-col
            v-for="img in generated_images"
            :key="img"
            style="margin-top: 80px"
            md="6"
            lg="4"
            xl="3"
          >
            <center>
              <ImageItem
                :app_state="app_state"
                :path="img"
                :style_obj="{ 'max-width': '85%' }"
              ></ImageItem>
            </center>
          </b-col>
        </b-row>
        <br />
      </div>

      <div v-if="backend_error" style="color: red; margin-top: 50px">
        <div class="center loader_box">
          <p>Error: {{ backend_error }}</p>
        </div>
      </div>
    </div>

    <div
      v-if="
        !stable_diffusion.is_input_avail &&
        stable_diffusion.generated_by == 'txt2img'
      "
    >
      <LoaderModal
        :loading_percentage="done_percentage"
        loading_title="생성중..."
        :loading_desc="stable_diffusion.generation_state_msg"
      ></LoaderModal>
    </div>

    <div class="bottom_float">
      <p>빠른 작업을 위해 다른 작업들을 꺼주세요.</p>
    </div>

    <div
      @click="share_current_arthub"
      v-if="generated_images.length > 0"
      class="l_button bottom_float"
      style="
        right: 10px;
        bottom: 15px;
        background-color: inherit;
        cursor: pointer;
      "
    >
      ArtHub.ai에 공유하기
    </div>
  </div>
</template>
<script>
import LoaderModal from "../components_bare/LoaderModal.vue";
import Vue from "vue";
import ImageItem from "../components/ImageItem.vue";
import { share_on_arthub } from "../utils.js";
import SDOptionsDropdown from "../components_bare/SDOptionsDropdown.vue";

const GOOGLE_API_KEY = "SETTHISVALUE";
const ART_PROMPT = [
  "anthony jones, Loish, painterly style by Gerald parel, craig mullins, marc simonetti, mike mignola, flat colors illustration, bright and colorful, high contrast, Mythology, cinematic, detailed, atmospheric, epic , concept art, Matte painting, Lord of the rings, Game of Thrones, shafts of lighting, mist, , photorealistic, concept art, volumetric light, cinematic epic + rule of thirds | 35mm| octane render, 8k, corona render, movie concept art, octane render, 8k, corona render, cinematic, trending on artstation, movie concept art, cinematic composition , ultra detailed, realistic , hiperealistic , volumetric lighting , 8k",
  "cinematic, raining, night time, detailed, epic , concept art, Matte painting, shafts of lighting, mist, photorealistic, concept art, volumetric light, cinematic epic + rule of thirds, movie concept art, 8k, cinematic, trending on artstation, movie concept art, cinematic composition , ultra detailed, realistic , hyper realistic , volumetric lighting , 8k",
  "rendered in octane, in the style of Luc Schuiten, craig mullins, solarpunk in deviantart, photorealistic, highly detailed, Vincent Callebaut, elena of avalor, highly detailed",
  "d & d concept art, d & d wallpaper, warm, digital art. art by james gurney and larry elmore",
  "Greg Rutkowski, Zabrocki, Karlkka, Jayison Devadas, Phuoc Quan, trending on Artstation, 8K, ultra wide angle, pincushion lens effect.",
  "Moebius, Greg Rutkowski, Zabrocki, Karlkka, Jayison Devadas, Phuoc Quan, trending on Artstation, 8K, ultra wide angle, pincushion lens effect.",
  "beautiful bright colors, hypermaximalist, futuristic, cyberpunk setting, luxury, elite, cinematic, techwear fashion, Errolson Hugh, Sacai, Nike ACG, Yohji Yamamoto, Y3, ACRNYM, matte painting",
  "in the style of artgerm, and wlop, chanel jewelry, cinematic lighting, hyperdetailed, 8 k realistic, symmetrical, global illumination, radiant light, love and mercy, frostbite 3 engine, cryengine, dof, trending on artstation, digital art, crepuscular ray",
  "disney infinity 3 star wars style, volumetric lighting, subsurface scattering, photorealistic, octane render, medium shot, studio ghibli, pixar and disney animation, sharp, rendered in unreal engine 5, anime key art by greg rutkowski and josh black, bloom, dramatic lighting",
  "medium shot, studio ghibli, pixar and disney animation, sharp, rendered in unreal engine 5, anime key art by greg rutkowski, bloom, dramatic lighting",
  "digital portrait, concept art, illustration, natural soft rim light, anatomical, facial muscles, elegant, regal, hyper realistic, ultra detailed, 0 6 0 8 wear techwear clothing, octane render, darriel diano style, volumetric lighting, 8 k post – production, artstation hq, unreal engine 5, unity engine",
  "highly saturated colors, fantasy character, detailed illustration, hd, 4k, digital art, overdetailed art, concept art, Dan Mumford, Krzysztof Maziarz, trending on artstation",
  "beautiful painting, detailed illustration, digital art, overdetailed art, concept art, full character, character concept, long hair, full body shot, highly saturated colors, fantasy character, detailed illustration, hd, 4k, digital art, overdetailed art, concept art, Dan Mumford, Greg rutkowski, Victo Ngai",
  "16K resolution, Landscape veduta photo by Dustin Lefevre & tdraw, 8k resolution, detailed landscape painting by Ivan Shishkin, DeviantArt, Flickr, rendered in Enscape, Miyazaki, Nausicaa Ghibli, Breath of The Wild, 4k detailed post processing, atmospheric, hyper realistic, 8k, epic composition, cinematic, artstation",
  "ultra wide shot, atmospheric, hyper realistic, 8k, epic composition, cinematic, octane render, artstation landscape vista photography by Carr Clifton & Galen Rowell, 16K resolution, Landscape veduta photo by Dustin Lefevre & tdraw, 8k resolution, detailed landscape painting by Ivan Shishkin, DeviantArt, Flickr, rendered in Enscape, Miyazaki, Nausicaa Ghibli, Breath of The Wild, 4k detailed post processing, artstation, rendering by octane, unreal",
  "hyper realistic, 8k, epic composition, cinematic, octane render, artstation landscape vista photography by Carr Clifton & Galen Rowell, 16K resolution, Landscape veduta photo by Dustin Lefevre & tdraw, 8k resolution, detailed landscape painting by Ivan Shishkin, DeviantArt, Flickr, rendered in Enscape, Miyazaki, Nausicaa Ghibli, Breath of The Wild, 4k detailed post processing, artstation, rendering by octane, unreal engine",
  "stranded alone and roaming in the chaos across a depressing abandoned post – apocalyptic landscape, post – apocalyptic corrupted themes, artstation trending, beautiful art landscape, detailed simon stalenhag landscape",
  "atmospheric, hyper realistic, epic composition, cinematic, landscape vista photography by Carr Clifton & Galen Rowell, 16K resolution, Landscape veduta photo by Dustin Lefevre & tdraw, detailed landscape painting by Ivan Shishkin, DeviantArt, Flickr, rendered in Enscape, Miyazaki, Nausicaa Ghibli, Breath of The Wild, 4k detailed post processing, artstation, unreal engine",
  "atmospheric, hyper realistic, 8k, epic composition, cinematic, octane render, artstation landscape vista photography by Carr Clifton & Galen Rowell, 16K resolution, Landscape veduta photo by Dustin Lefevre & tdraw, 8k resolution, detailed landscape painting by Ivan Shishkin, DeviantArt, Flickr, rendered in Enscape, Miyazaki, Nausicaa Ghibli, Breath of The Wild, 4k detailed post processing, artstation, rendering by octane, unreal engine",
  "art station, landscape, concept art, illustration, highly detailed artwork cinematic, hyper realistic painting",
  "3d with depth of field, blurred background. female. nautilus. A highly detailed epic cinematic concept art CG render. made in Blender and Photoshop, octane render, excellent composition, cinematic dystopian brutalist atmosphere. dynamic lighting. dramatic lighting. cinematic lighting. aesthetic. stylized. very inspirational. detailed. hq. realistic. warm light. vibrant color scheme. highly detailed. muted colors. Moody. Filmic.",
  "ctane render, 8 k, exploration, cinematic, trending on artstation, by beeple, realistic, 3 5 mm camera, unreal engine, hyper detailed, photo – realistic maximum detai, volumetric light, moody cinematic epic concept art, realistic matte painting, hyper photorealistic, concept art, volumetric light, cinematic epic, octane render, 8 k, corona render, movie concept art, octane render, 8 k, corona render, cinematic, trending on artstation, movie concept art, cinematic composition, ultra – detailed, realistic, hyper – realistic, volumetric lighting, 8 k",
  "WLOP, Stanley Artgerm Lau, Ruan Jia and Fenghua Zhong, trending on ArtStation, subtle muted cinematic colors, made in Maya, Blender and Photoshop, octane render, excellent composition, cinematic atmosphere, dynamic dramatic cinematic lighting, precise correct anatomy, aesthetic, very inspirational, arthouse",
];

export default {
  name: "ImgGenerate",
  props: {
    app_state: Object,
    stable_diffusion: Object,
  },
  components: { LoaderModal, ImageItem, SDOptionsDropdown },
  mounted() {},
  data() {
    return {
      img_w: 768,
      img_h: 512,
      dif_steps: 25,
      guidence_scale: 7.5,
      is_adv_options: false,
      seed: "",
      prompt: "",
      num_imgs: 1,
      batch_size: 1,
      generated_images: [],
      backend_error: "",
      done_percentage: -1,
      is_stopping: false,
      modifiers: require("../modifiers.json"),
      is_negative_prompt_avail: false,
      is_random_art_style_avail: true,
      is_translation_avail: true,
      negative_prompt: "",
      selected_model: "Default",
    };
  },
  computed: {
    this_object() {
      return this;
    },
  },
  methods: {
    async generate_from_prompt() {
      let seed = 0;
      if (this.seed) seed = Number(this.seed);
      else seed = Math.floor(Math.random() * 100000);

      this.computed_seed = seed;

      let params = {
        prompt: this.prompt,
        W: Number(this.img_w),
        H: Number(this.img_h),
        seed: seed,
        scale: this.guidence_scale,
        ddim_steps: this.dif_steps,
        num_imgs: this.num_imgs,
        batch_size: this.batch_size,
      };

      if (
        this.selected_model &&
        this.selected_model != "Default" &&
        this.app_state.app_data.custom_models[this.selected_model]
      ) {
        params.model_id = -1;
        params.custom_model_path =
          this.app_state.app_data.custom_models[this.selected_model].path;
      }

      if (this.is_negative_prompt_avail) {
        params["negative_prompt"] = this.negative_prompt;
      }

      if (this.is_translation_avail) {
        this.translated_prompt = await this.translate(this.prompt);
        this.translated_negative_prompt = await this.translate(
          this.negative_prompt
        );
        params["prompt"] = this.translated_prompt;
        params["negative_prompt"] = this.translated_negative_prompt;
      }

      if (this.is_random_art_style_avail) {
        this.selected_art_prompt =
          ART_PROMPT[Math.floor(Math.random() * ART_PROMPT.length)];
        params["prompt"] += ", " + this.selected_art_prompt;
      }

      let that = this;

      if (this.prompt.trim() == "") {
        Vue.$toast.default("You need to enter a prompt to generate an image");
        return;
      }

      this.backend_error = "";
      Vue.set(this, "generated_images", []);
      this.done_percentage = -1;

      let history_key = Math.random();

      let callbacks = {
        on_img(img_path) {
          that.generated_images.push(img_path);

          if (!that.app_state.app_data.history[history_key]) {
            let p = {
              prompt: that.prompt,
              seed: seed,
              img_w: that.img_w,
              img_h: that.img_h,
              key: history_key,
              imgs: [],
              guidence_scale: that.guidence_scale,
              dif_steps: that.dif_steps,
            };
            if (that.stable_diffusion.model_version)
              p["model_version"] = that.stable_diffusion.model_version;
            if (that.is_negative_prompt_avail)
              p["negative_prompt"] = that.negative_prompt;
            if (this.is_translation_avail) {
              p["prompt"] = that.translated_prompt;
              p["negative_prompt"] = that.translated_negative_prompt;
            }
            if (that.is_random_art_style_avail) {
              p["prompt"] += ", " + that.selected_art_prompt;
            }
            Vue.set(that.app_state.app_data.history, history_key, p);
          }

          that.app_state.app_data.history[history_key].imgs.push(img_path);

          console.log(that.app_state.app_data.history);
        },
        on_progress(p) {
          that.done_percentage = p;
        },
        on_err(err) {
          that.backend_error = err;
        },
      };

      this.is_stopping = false;

      if (this.stable_diffusion)
        this.stable_diffusion.text_to_img(params, callbacks, "txt2img");
    },

    open_arthub() {
      window.ipcRenderer.sendSync("open_url", "https://arthub.ai");
    },

    stop_generation() {
      this.is_stopping = true;
      this.stable_diffusion.interupt();
    },

    add_style(tag) {
      this.prompt += ", " + tag;
    },

    share_current_arthub() {
      this.app_state.global_loader_modal_msg = "Uploading";
      let params = {
        "Img Width": Number(this.img_w),
        "Img Height": Number(this.img_h),
        Seed: this.computed_seed,
        Scale: this.guidence_scale,
        Steps: this.dif_steps,
        model_version: this.stable_diffusion.model_version,
      };
      if (this.is_negative_prompt_avail)
        params["Negative Prompt"] = this.negative_prompt;

      let that = this;
      share_on_arthub(that.generated_images, params, that.prompt)
        .then(function () {
          that.app_state.global_loader_modal_msg = "";
        })
        .catch(function () {
          alert("Error in uploading.");
          that.app_state.global_loader_modal_msg = "";
        });
    },
    translate(prompt) {
      let translated_prompt = "";
      let url =
        "https://translation.googleapis.com/language/translate/v2?key=" +
        GOOGLE_API_KEY;
      let body = {
        q: prompt,
        source: "ko",
        target: "en",
        format: "text",
      };
      let options = {
        method: "POST",
        body: JSON.stringify(body),
        headers: {
          "Content-Type": "application/json",
        },
      };
      // generate fetch promise
      return new Promise((resolve, reject) => {
        fetch(url, options)
          .then((response) => response.json())
          .then((data) => {
            translated_prompt = data.data.translations[0].translatedText;
            console.log(translated_prompt);
            resolve(translated_prompt);
          })
          .catch((err) => {
            console.log(err);
            reject(prompt);
          });
      });
    },
  },
};
</script>
<style>
.center {
  margin: 0;
  position: absolute;
  top: 50%;
  left: 50%;
  -ms-transform: translate(-50%, -50%);
  transform: translate(-50%, -50%);
}

.disabled {
  pointer-events: none;
  opacity: 0.5;
}

.loader_box {
  padding: 20px;
  /*height: calc(160px);*/
  border-radius: 12px 12px 12px 12px;
}

.ad_form_box {
  float: left;
  background-color: rgba(0, 0, 0, 0.1);
  padding: 10px;
  margin-right: 10px;
  border-radius: 3px 3px 3px 3px;
}

.gal_img {
  border-radius: 12px 12px 12px 12px;
  /* background-color: white; */
  border-style: solid;
  border-width: 1px;
  border-color: rgba(0, 0, 0, 0.1);
  cursor: pointer;
  box-shadow: 0px 0px 1.76351px rgba(40, 41, 61, 0.04),
    0px 3.52703px 7.05405px rgba(96, 97, 112, 0.16);
}

@media (prefers-color-scheme: dark) {
  .gal_img {
    border-color: rgba(255, 255, 255, 0.3);
  }
}

.negative_prompt_tb {
  color: brown !important;
}

@media (prefers-color-scheme: dark) {
  .negative_prompt_tb {
    color: rgb(210, 0, 0) !important ;
  }
}
</style>
