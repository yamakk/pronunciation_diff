<template lang="pug">
  #app
    div Pronunciation Checker
    p
    div
      span(class="example") Try ...&nbsp;&nbsp;
      a(class="example" @click="original=examples['ezkiel']" href="#") Ezekiel 25:17
      | &nbsp;&nbsp;
      a(class="example" @click="original=examples['soramimi']" href="#") 空耳アワー
      | &nbsp;&nbsp;
      a(class="example" @click="original=examples['passport']" href="#") いずれパスポート取りたいんです
      | &nbsp;&nbsp;
      a(class="example" @click="original=examples['heike']" href="#") 祇園精舎
      | &nbsp;&nbsp;
    p
    //- p(class="example") or nytimes headlines
    //- span(v-for="item in nytimes")
    //-   | &nbsp;&nbsp;
    //-   a(class="example" @click="original=item.title" href="#") {{item.title}}
    p
    textarea(v-model="original", rows=7, cols=70)
    p(class="example") Select Language &nbsp;
      select(v-model="lang")
        option(v-for="_lang in languages" :value="_lang.value") {{_lang.text}}
    p
    button(class="button" @click="start_recognition" :style="{background: is_recording ? 'red' : '#ccc'}")
      p(v-if="!is_recording") start recording
      p(v-else) stop recording ■
    p
    div(v-if="is_recording")
      div.parts {{progress_transcript}}
    p
    div(v-if="is_recording_finished" class="transcript")
      span(v-for="word in diff.words")
        span(:style="{color:wordcolor(word)}") {{word.value}}
        //span(:style="{color: word.added ? 'red' : word.removed ? 'grey' : 'green' }") {{word.value}}
    div(v-if="is_recording_finished" class="example")
      span Your score is
      span(class="score") {{ this.diff.score.toFixed(0) }}
      span  out of 100

    //- span(v-for="word in diff.words")
    //-   li {{word}}

    //- p {{this.diff}}
    //div removed...{{diff.removed}} added...{{diff.added}}
     |   total...{{this.original.trim(' ').split(' ').length}} ok...{{diff.ok}} penalty..{{diff.penalty}}

    //div {{this.transcript}}

</template>

<script>
import * as differ from 'diff'
import axios from 'Axios'

export default {
  name: 'App',
  data () {
    return {
        nytimes: '',
        original:"Manners maketh man. Do you know what that means? Then, let me teach you a lesson",
        transcript: '',
        progress_transcript: '',
        lang: 'en-US',
        is_recording_finished: false,
        is_recording:false,
        recognition: new window.webkitSpeechRecognition(),
        languages:[{value:'en-US', text:'English US'}, {value:'en-GB', text:'English UK'},
                   {value:'cmn-Hans-CN',text:'普通话 (中国大陆)'},
                   {value:'ko-KR', text:'한국어'},{value:'fr-FR', text:'Français'}, {value:'de-DE', text:'Deutsch'},
                   {value:'ru-RU', text:'Pусский'}, {value:'sv-SE', text:'Svenska'},
                   {value:'vi-VN', text:'Tiếng Việt'}, {value:'ja-JP', text:'日本語'} ],
        examples:
          { ezkiel: "The path of the righteous man is beset on all sides by the inequities of the selfish and the tyranny of evil men. Blessed is he, who in the name of charity and good will, shepherds the weak through the valley of darkness, for he is truly his brother's keeper and the finder of lost children. And I will strike down upon thee with great vengeance and furious anger those who would attempt to poison and destroy my brothers. And you will know my name is the Lord when I lay my vengeance upon thee.",
            burn_nortice: "My name is Michael Westen. I used to be a spy until... We’ve got a burn notice on you. You’re blacklisted. When you’re burned, you’ve got nothing. No cash, no credit, no job history. You’re stuck in whatever city they decide to dump you in. You do whatever work comes your way. You rely on anyone who’s still talking to you. A trigger happy ex-girlfriend. An old friend who used to inform on you to the FBI. Family too. If you’re desperate. Bottom line, as long as you’re burned, you’re not going anywhere.",
            soramimi: "誰が言ったか知らないが言われてみれば確かに聞こえる空耳アワーのお時間です",
            passport: "いずれパスポート取りたいんです",
            heike:"祇園精舎の鐘の声、諸行無常の響きあり。沙羅双樹の花の色、 盛者必衰の理をあらわす。驕れる人も久しからず、ただ春の夜の夢のごとし。猛き者もついにはほろびぬ、ひとえに風の前の塵に同じ。"


          }
      }
  },
  mounted(){
    if(!this.recognition){
      alert('Your browser is not supported. Play with Chrome for Mac/Win')
    }
    // axios.get("https://api.nytimes.com/svc/news/v3/content/all/all.json?api-key=Wd8AXKWgAYaSJylsrmhCCCCQJasoApl2")
    //  .then(response => {this.nytimes = response.data.results.slice(0,10)})
  },
  methods:{
    start_recognition(){
      if( this.is_recording ){
        this.stop_recognition()
      }else{
        this.transcript = ''
        this.is_recording = true
        this.is_recording_finished = false
        this.recognition.lang = this.lang
        this.recognition.interimResults = true
        this.recognition.continuous = true
        this.recognition.start()
        this.recognition.onresult = (event) => {
          for (let i = event.resultIndex; i < event.results.length; i++) {
            let _input = event.results[i][0].transcript;
            if (event.results[i].isFinal) {
              this.transcript += _input;
            } else {
              this.progress_transcript = _input;
            }
          }
        }
      }
    },
    stop_recognition(){
      //console.log('stop')
      this.recognition.stop()
      this.is_recording = false
      this.is_recording_finished = true
    },
    wordcolor(word){
      //console.log(word)
      if(word.added) {
        return 'red'
      }else if(word.removed){
        if(word.value.match(/ $/)){
          return '#666'
        }else{
          return 'gray'
        }
      }else{
        return 'green'
      }
    },
    get_diff_type(lang){
      if(['ja-JP', 'cmn-Hans-CN','ko-KR'].includes(this.lang)){
        return 'chars'
      }else{
        return 'words'
      }
    }
  },
  computed:{
    diff(){
      let _original = this.original.replace(/[\.,\?\!、|。|！|？]/g,' ').replace(/’/g,"'").replace(/ {2,}/g, ' ')
      let words = null;
      let total = 0;
      if(this.get_diff_type(this.lang)=='words'){
        words = differ.diffWords(_original, this.transcript, {ignoreCase:true})
        total = _original.trim(' ').split(' ').length
      }else{
        words = differ.diffChars(_original, this.transcript)
        total = _original.trim(' ').length
      }
      let removed = 0
      let added = 0
      let penalty = 0
      let ok = 0
      let red=0
      let blue=0
      let gray=0
      let score = 0
      let lastword = words[words.length-1]
      for(let word of words){
        if(word.value.trim() != ''){
          if(this.get_diff_type(this.lang)=='words'){
            removed += word.removed ? word.value.trim().split(' ').length : 0
            added += word.added ? word.value.trim().split(' ').length : 0
            ok += !word.added && !word.removed ? word.value.trim().split(' ').length : 0
            if(word == lastword && word.removed){
              word.value = word.value += ' '
            }
            red  += this.wordcolor(word)=='red' ? word.value.trim().split(' ').length : 0
            blue += this.wordcolor(word)=='#666' ? word.value.trim().split(' ').length : 0
            penalty = red + blue
          }else{
            removed += word.removed ? word.value.trim().length : 0
            added += word.added ? word.value.trim().length : 0
            ok += !word.added && !word.removed ? word.value.trim().length : 0
            if(word == lastword && word.removed){
              word.value = word.value += ' '
            }
            red  += this.wordcolor(word)=='red' ? word.value.trim().length : 0
            blue += this.wordcolor(word)=='#666' ? word.value.trim().length : 0
            gray += this.wordcolor(word)=='gray' ? word.value.trim().length : 0
            penalty = red + blue + gray
          }
          // word['rv'] =  word.removed ? word.value.trim().split(' ').length : 0
          // word['av '] = word.added ? word.value.trim().split(' ').length : 0
          // //赤と青を数える
          // 最後のワードはスペースがつかない
        }
      }

      //score = (ok - penalty) / total * 100
      score = (total - penalty) / total * 100
      score = score > 0 ? score : 0
      return {words:words, added:added, removed:removed, ok:ok, total:total, penalty:penalty, score:score}
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
  font-size: 2em;
}
.example {font-size:10px; color:#666;}
.button {  font-size:20px; border-radius: 5px; }
.parts {color:#999;}
.transcript { padding:40px;}
textarea {padding:10px; font-size:18px;}
.score { font-style: bold; color:green; font-size:50px;}
</style>
