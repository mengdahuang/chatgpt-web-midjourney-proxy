<script setup lang="ts">
//boy, Cyberpunk , Top view , Face Shot (VCU) , Warm light  --style raw  --ar 3:4 --q 0.5 --v 5.2
import { ref,computed,watch,onMounted } from "vue";
import config from "./draw.json";
import {  NSelect,NInput,NButton,NTag,NPopover, useMessage,NDivider} from 'naive-ui'; 
import {  SvgIcon } from '@/components/common'
import { useBasicLayout } from '@/hooks/useBasicLayout'
const { isMobile } = useBasicLayout()
import AiMsg from './aiMsg.vue' 
//import aiFace from './aiFace.vue' 
import { mlog, train, upImg ,getMjAll } from '@/api' 
//import {copyText3} from "@/utils/format";
import { homeStore ,useChatStore} from "@/store";
const chatStore = useChatStore()
import {t} from "@/locales"
//import { upImg } from "./mj";

const vf=[{s:'width: 100%; height: 100%;',label:'1:1'}
,{s:'width: 100%; height: 75%;',label:'4:3'}
,{s:'width: 75%; height: 100%;',label:'3:4'}
,{s:'width: 100%; height: 50%;',label:'16:9'}
,{s:'width: 50%; height: 100%;',label:'9:16'}
 ];

const f=ref({bili:-1, quality:'',view:'',light:'',shot:'',style:'', styles:'',version:'--v 5.2'});
const st =ref({text:'',isDisabled:false,isLoad:false
    ,fileBase64:[],bot:'',showFace:false
});
const farr= [
{ k:'style',v:t('mjchat.tStyle') }
,{ k:'view',v: t('mjchat.tView') }
,{ k:'shot',v: t('mjchat.tShot') }
,{ k:'light',v: t('mjchat.tLight') }
,{ k:'quality',v: t('mjchat.tQuality') }
,{ k:'styles',v:t('mjchat.tStyles') }
,{ k:'version',v:t('mjchat.tVersion') }
 ];

const msgRef = ref()
const fsRef= ref() 
const fsRef2 = ref()
const $emit=defineEmits(['drawSent','close']);
const props = defineProps({buttonDisabled:Boolean});

const isDisabled = computed(() => {
    return props.buttonDisabled || st.value.isLoad || st.value.text.trim()==''
})
const ms=   useMessage();
function create( ){
   
   
    st.value.isLoad=true
    train( st.value.text.trim()).then(ps=>{
        const rz={ prompt: st.value.text.trim() , drawText: createPrompt( ps) }
        if( ps  ) drawSent(rz)
        st.value.text=''
        st.value.isLoad=false
    }).catch(err=>{
        msgRef.value.showError(err)
        st.value.isLoad=false
    })

    
} 
function drawSent(rz:any){
    let rz2= rz;
    if(st.value.fileBase64) {
        rz2.fileBase64=st.value.fileBase64
    }
    if( st.value.bot=='NIJI_JOURNEY' ){
        rz2.bot='NIJI_JOURNEY';
    }
    $emit('drawSent', rz2 )
    st.value.fileBase64= [];
}
function createPrompt(rz:string){
    if( rz =='') {
        msgRef.value.showError(t('mjchat.placeInput') );
        return '';
    }
     
   
    for(let v of farr){
        if( ! f.value[v.k] || f.value[v.k]==null || f.value[v.k]=='' ) continue;
         mlog('k ', rz,  f.value  );
        if(v.k=='quality') rz +=`  --q ${f.value.quality}`;
        else if(v.k=='styles') { if( f.value.styles ) rz +=` ${f.value.styles}`;}
        else if(v.k=='version') {
            st.value.bot= '';
           if(['MID_JOURNEY','NIJI_JOURNEY'].indexOf(f.value.version)>-1 ){
                 st.value.bot= f.value.version ;
           } else   rz +=` ${f.value.version}`;
        }
        else if( f.value[v.k] ) rz +=` , ${f.value[v.k]}`;
    }
    mlog('createPrompt ', rz,  f.value  );
    if(f.value.bili>-1) rz +=` --ar ${vf[f.value.bili].label}`;
    return rz ;
}
 
// const copy=()=>{
//     copyText3( '哦们sd').then(()=>msgRef.value.showMsg('复制成功345！'));
// }
// const copy2= ()=>{
//     copyRef.value.click();
// }
function selectFile(input:any){
    if(st.value.fileBase64.length>=5 ) {
        ms.error( t('mjchat.more5sb'));
        return;
    }
    upImg(input.target.files[0]).then(d=>st.value.fileBase64.push(d) ).catch(e=>msgRef.value.showError(e));
    
}

//图生文
function selectFile2(input:any){
     
    upImg(input.target.files[0]).then(d=>{
        mlog('f2base64>> ',d );
        let obj={
            action:'img2txt', 
            data:{
                "base64":d
                ,"botType": "MID_JOURNEY"
            }
        }
        homeStore.setMyData({act:'draw',actData:obj});
        //input.value.value='';
        fsRef2.value.value='';
       
    })
    .catch(e=>msgRef.value.showError(e))
}

const same2=()=>{
     st.value.text= homeStore.myData.actData.prompt;
    f.value.version='';
    f.value.quality='';
}
watch(()=>homeStore.myData.act,(n)=>{
   // n=='copy' && copy2();
    n=='same2' && same2();
});
onMounted(()=>{
    homeStore.myData.act=='same2' && same2();
});



const exportToTxt= async ()=>{
    let txtContent ='';
    mlog('sss',txtContent,chatStore.$state.chat.length  );
   
    let d = await getMjAll( chatStore.$state);
    if(d.length==0) {
        //ms.info('暂时没作品');
        ms.info( t('mjchat.noproducet'));
        return;
    }
    d.forEach((v:Chat.Chat,i:number)=>{
        if( v.opt&& v.opt?.status=='SUCCESS' && v.opt?.imageUrl ) {
                txtContent += v.opt?.imageUrl+ "\n\n";
        }
    })
    if(txtContent=='') {
         ms.info( t('mjchat.noproducet'));
        return;
    }
    let blob = new Blob([txtContent], { type: "text/plain" });
    let a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = t('mjchat.downloadSave') ;
    a.click();
    ms.success( t('mjchat.exSuccess'));
}
//const config=
</script>
<template>
<AiMsg ref="msgRef" />
<input type="file"  @change="selectFile"  ref="fsRef" style="display: none" accept="image/jpeg, image/jpg, image/png, image/gif"/>
<input type="file"  @change="selectFile2" ref="fsRef2" style="display: none" accept="image/jpeg, image/jpg, image/png, image/gif"/>

<div class="overflow-y-auto bg-[#fafbfc] px-4 dark:bg-[#18181c] h-full ">
    
    <section class="mb-4">
        <div class="mr-1  mb-2 flex justify-between items-center">
            <div class="text-sm">{{ $t('mjchat.imgBili') }}</div>
            <div>
            <NPopover trigger="hover">
                <template #trigger>
                 <SvgIcon icon="iconoir:database-export" class="text-lg cursor-pointer" @click="exportToTxt"></SvgIcon>
                </template>
                <div>{{ $t('mjchat.imagEx') }}</div>
            </NPopover>
            </div>
        </div>
        <div class=" flex items-center justify-between space-x-1">
            <template  v-for="(item,index) in vf" >
            <section class="aspect-item flex-1 rounded border-2 dark:border-neutral-700 cursor-pointer"  :class="{'active':index==f.bili}"  @click="f.bili=index">
                <div class="aspect-box-wrapper mx-auto my-2 flex h-5 w-5 items-center justify-center">
                    <div class="aspect-box rounded border-2 dark:border-neutral-700" :style="item.s"></div>
                </div>
                <p class="mb-1 text-center text-sm">{{ item.label }}</p>
            </section>
            </template>
             
        </div>
    </section>
    <section class="mb-4 flex justify-between items-center" v-for=" v in farr">
     <div>{{ v.v }}</div>
     
    <n-select v-model:value="f[v.k]" :options="config[v.k+'List']" size="small"  class="!w-[60%]" :clearable="true" />
    </section>
    <!-- <section class="mb-4 flex justify-between items-center"  >
     <div>机器人</div>
    <n-select v-model:value="st.bot" :options="config.botList" size="small"  class="!w-[60%]" :clearable="true" />

     </section> -->
    <div class="mb-1">
     <n-input    type="textarea"  v-model:value="st.text"   :placeholder="$t('mjchat.prompt')" round clearable maxlength="500" show-count 
      :autosize="{   minRows:2, maxRows:5 }" />
    </div>
    <div class="mb-4 flex justify-between items-center">
        <div class="flex justify-start items-center">
             <NPopover trigger="hover">
                <template #trigger>
                <n-tag type="error" round size="small" style="cursor: pointer; " :bordered="false" @click="fsRef.click()"   v-if="st.fileBase64.length">
                <div style="display: flex;">  <SvgIcon icon="mdi:file-chart-check-outline" /> {{ $t('mjchat.imgCYes') }} </div>
                </n-tag>
                <n-tag type="warning" round size="small" style="cursor: pointer; " :bordered="false" @click="fsRef.click()"   v-else="st.fileBase64">
                <div style="display: flex;">  <SvgIcon icon="mdi:file-document-plus-outline" />  {{ $t('mjchat.imgCUpload') }} </div>
                </n-tag>
                </template>
                <div  style="max-width: 240px;">
                <p v-html="$t('mjchat.imgCInfo')"></p>
                
                3.<a class="text-green-500 cursor-pointer"  @click="fsRef.click()" v-html="$t('mjchat.imgCadd')"></a><br/> 
                <div  v-if="st.fileBase64.length>0" class="flex justify-start items-baseline">
                    <div class="p-1" v-for="(v ) in st.fileBase64">
                        <img  class="w-[60px]" :src="v">
                        <br/> 
                        <NButton size="small" @click="st.fileBase64= st.fileBase64.filter((item)=>item!=v) " type="warning" >{{$t('mjchat.del')}}</NButton>
                    </div>
                    
                </div>
                </div>
             </NPopover>

             <div class="pl-1">
               <NPopover trigger="hover">
                    <template #trigger>
                        <n-tag type="warning" round size="small" style="cursor: pointer; " :bordered="false" @click="fsRef2.click()"    >
                            <div style="display: flex;">  <SvgIcon icon="fluent:image-edit-16-regular" />  {{$t('mjchat.img2text')}} </div>
                        </n-tag>
                    </template>
                     <div  style="max-width: 240px;" v-html="$t('mjchat.img2textinfo')">
                     </div>
                </NPopover>
            </div>
        </div>
        

        <div class="flex "  v-if="$t('mjchat.imgcreate').indexOf('生成图片')!==-1">
         <n-button type="primary" :block="true" :disabled="isDisabled"  @click="create()">
            <SvgIcon icon="mingcute:send-plane-fill" />  
            
            <template v-if="st.isLoad">{{$t('mjchat.traning')}} </template>
            <template v-else> {{$t('mjchat.imgcreate')}}</template>
            
        </n-button>
        </div>

        
    </div>

    <div class="flex "  v-if="$t('mjchat.imgcreate').indexOf('生成图片')==-1">
        <n-button type="primary" :block="true" :disabled="isDisabled"  @click="create()">
        <SvgIcon icon="mingcute:send-plane-fill" />  
        
        <template v-if="st.isLoad">{{$t('mjchat.traning')}} </template>
        <template v-else> {{$t('mjchat.imgcreate')}}</template>
        
    </n-button>
    </div>

    <!-- <div>
    <NDivider dashed>
        <NTag type="success" round ><div class="cursor-pointer" @click="st.showFace= !st.showFace">换脸服务</div></NTag>
    </NDivider>
    <aiFace v-if="st.showFace" />
    </div> -->
    <!-- <div class="mb-4 flex justify-between items-center">
        <div @click="copy()" ref="copyRef">复制</div>
        <div @click="copy2()"  >复制2</div> 
    </div> -->

   <ul class="pt-4"  v-if="!isMobile" v-html="$t('mjchat.imginfo')">
    
</ul>

</div>



</template>
<style>
    .aspect-item.active, .aspect-item.active .aspect-box{
        border-color:#86dfba ;
         
    }
</style>