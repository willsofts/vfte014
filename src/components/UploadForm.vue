<template>
  <DialogForm ref="dialogForm" :id="dialogId">
    <template v-slot:header>
      <h4 class="modal-title" v-if="jsonMode">{{ labels.title_upload_json }}</h4>
      <h4 class="modal-title" v-if="excelMode">{{ labels.title_upload_excel }}</h4>
    </template>
    <template v-slot:default>
      <form ref="uploadform" id="uploadform" name="uploadform">
        <div class="row row-height">
          <div class="col-height col-md-5">
            <label for="labelid">{{ labels.labelid_label }}</label>
            <div class="input-group has-validation" :class="{'has-error': v$.labelid.$error}">
              <select ref="labelid" id="labelid" name="labelid" v-model="localData.labelid" class="form-control input-md">
                <option v-for="item in dataCategory.tprog" :key="item.id" :value="item.id">{{item.text}}</option>
              </select>
              <label class="required">*</label>
            </div>
            <span v-if="v$.labelid.$error" class="has-error">{{ v$.labelid.$errors[0].$message }}</span>
          </div>
        </div>
        <div class="row row-height">
          <div class="col-height col-md-8">
            <label for="filename">{{ labels.filename_label }}</label>
            <div class="input-group has-validation" :class="{'has-error': v$.filename.$error}">
              <input ref="filename" type="file" id="filename" name="filename" class="form-control input-md" :accept="accept" @change="fileChanged"/>
              <label class="required">*</label>
            </div>
            <span v-if="v$.filename.$error" class="has-error">{{ v$.filename.$errors[0].$message }}</span>
          </div>
        </div>
      </form>
    </template>
    <template v-slot:footer>
      <button ref="uploadbutton" id="uploadbutton" class="btn btn-dark btn-sm" @click="uploadClick"><em class="fa fa-upload fa-btn-icon"></em>{{ labels.upload_button }}</button>
      <button class="btn btn-dark btn-sm" data-dismiss="modal"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
    </template>
  </DialogForm>
</template>
<style>
#uploadbutton { min-width: 100px; }
</style>
<script>
import { ref, computed, watch } from 'vue';
import { useVuelidate } from '@vuelidate/core';
import { required, helpers } from '@vuelidate/validators';
import $ from "jquery";
import { getBaseUrl, disableControls }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure, detectErrorResponse }  from '@willsofts/will-app';
import { confirmImport, successbox, serializeParameters } from '@willsofts/will-app';
import DialogForm from './DialogForm.vue';

const defaultData = {
  labelid: "index.xml",
  filename: "",
};

export default {
  components: {
    DialogForm
  },
  props: {
    modes: Object,
    labels: Object,
    dataCategory: Object,
    fileType: {
      type: String,
      default: "json",
    },
    accept: {
      type: String,
      default: "json",
    },
    dialogId: {
      type: String,
      default: "modaldialog_layerupload",
    },
  },
  emits: ["data-uploaded"],
  setup(props) {
    const localData = ref({ ...defaultData }); 
    const reqalert = ref(props.labels.empty_alert);
    const requiredMessage = () => {
      return helpers.withMessage(reqalert, required);
    }
    const validateRules = computed(() => { 
      return {
        labelid: { required: requiredMessage() },
        filename: { required: requiredMessage() },
      } 
    });
    const v$ = useVuelidate(validateRules, localData, { $lazy: true, $autoDirty: true });
    return { v$, localData, reqalert };
  },
  created() {
    watch(this.$props, (newProps) => {      
      this.reqalert = newProps.labels.empty_alert;
    });
  },
  computed: {
    jsonMode() {
      return this.$props.fileType=="json";
    },
    excelMode() {
      return this.$props.fileType=="excel";
    },
  },
  mounted() {
    this.$nextTick(() => {
      $("#"+this.$props.dialogId).find(".modal-dialog").draggable();
    });
  },
  methods: {
    reset(newData) {
      if(newData) this.localData = {...newData};
    },
    async uploadClick() {
      console.log("click: upload");
      disableControls($("#uploadbutton"));
      let valid = await this.validateForm();
      if(!valid) return;
      this.startUploadRecord();
    },
    async validateForm(focusing=true) {
      const valid = await this.v$.$validate();
      console.log("validate form: valid",valid);
      console.log("error:",this.v$.$errors);
      if(!valid) {
        if(focusing) {
          this.focusFirstError();
        }
        return false;
      }
      return true;
    },
    focusFirstError() {
      if(this.v$.$errors && this.v$.$errors.length > 0) {
        let input = this.v$.$errors[0].$property;
        let el = this.$refs[input];
        if(el) el.focus(); //if using ref
        else $("#"+input).trigger("focus"); //if using id
      }
    },
    showDialog(callback) {
      if(callback) $(this.$refs.dialogForm.$el).on("shown.bs.modal",callback);
      $(this.$refs.dialogForm.$el).modal("show");
    },  
    hideDialog() {
      $(this.$refs.dialogForm.$el).modal("hide");
    },
    resetRecord() {
      //reset to default data 
      this.reset(defaultData);
      //reset validator
      this.v$.$reset();
      //enable key field
    },
    startUpload() {
      this.resetRecord();
      this.showDialog(() => { this.$refs.labelid.focus(); });
    },
    startUploadRecord() {
      confirmImport(() => {
        this.uploadRecord(this.localData);
      });
    },
    uploadRecord(dataRecord) {
        let aform = document.getElementById("uploadform");
        let fd = new FormData(aform);
        let jsondata = {ajax: true};
        let formdata = serializeParameters(jsondata,dataRecord);
        startWaiting();
        $.ajax({
          url: getBaseUrl()+'/upload/sfte014/uploader',
          data: fd,
          headers: formdata.headers,
          type: "POST",
          dataType: "json",
          enctype: "multipart/form-data",
          processData: false, 
          contentType: false,
          error : function(transport,status,errorThrown) {
            console.error("error: status",status,"errorThrown",errorThrown);
            submitFailure(transport,status,errorThrown);
          },
          success: (data) => {
            stopWaiting();
            console.log("success",data);
            if(detectErrorResponse(data)) return;
            successbox(() => {
              this.hideDialog();
              this.$emit('data-uploaded',dataRecord,data);
            });
          }
      });
    },
    fileChanged() {
      this.localData.filename = this.$refs.filename.files[0];
    }
  }
};
</script>
