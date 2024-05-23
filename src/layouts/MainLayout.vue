<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated class="bg-color-primary">
      <q-toolbar>
        <q-toolbar-title class="text-center"> 
          DigiTiket 
        </q-toolbar-title>
        </q-toolbar>
    </q-header>
    <q-page-container class="absolute-center">
      <div class="row" >
        <div v-if="cameraStart" class="col-12  text-center"  >
          <video  autoplay height="80%"  ref="videoplay"></video>
          <q-btn  label="Tomar Foto" icon="photo_camera" @click="takePic()"/>
          <q-btn  label="Girar Cámara" icon="loop" @click="front=!front;usarCamera()"/>
        </div>
        <div class="col-12"> 
          <q-btn v-if="!cameraStart" @click="usarCamera()" size="45px" color="blue-grey-3" class="justify-center" >
            <q-icon name="photo_camera" size="90px"/>
            <br>
            <div class="text-weight-bold text-caption">Pulse para capturar imagen</div>
          </q-btn>
        </div>
      </div>
     
    </q-page-container>
    
    <q-dialog v-model="expanded"  >
      <q-card style="width: 100vw">
        <q-card-section class="row bg-primary items-center text-white">
          <div class="text-h6">Enviar por ...</div>
          <q-space />
          <q-btn flat icon="close" class="items-right text-white" @click="expanded = false"/>
        </q-card-section>
         <q-card-section>
          <!--q-input type="email" label="De" v-model="datosImagen.from"></q-input-->
          <!--q-input clearable type="email" label="Para" v-model="datosImagen.to">></q-input--->
          <q-input clearable label="Asunto" v-model="datosImagen.subject">></q-input>
          <q-input clearable text-area label="Texto" v-model="datosImagen.body" autogrow @keyup.enter.stop> ></q-input>
           <q-space />
        </q-card-section>
        <q-card-section> 
          <div class="justify-center text-center">
            <q-btn @click="sendMail()" icon-right="send" label="ENVIAR" color="primary"/>
          </div>
        </q-card-section>
      </q-card>
    </q-dialog>
  </q-layout>
</template>

<script>
import { date } from 'quasar'
import { jsPDF } from "jspdf"

export default {
  data(){ 
    return { 
      options: {},
      expanded: false,
      cameraStart: false,
      imageCapture: null,
      track: null,
      front: false,
      doc: new jsPDF(),
      datosImagen: { 
        // from: 'jvilata@vidawm.com',
        to: 'jvilata@edicom.es',
        subject: `digiticket-${date.formatDate(new Date(),'YYYY-MM-DD HHmmss')}`,
        body: '',
        data: '',
        dataB64: '',
        dataURL: '',
        oMyBlob: null
      }
    }
  },
  methods: { 
    usarCamera() {
      if (this.track!==null) this.track.stop()
      var props = {}
      if (this.front) {
        props = { video: true }
      } else {
        props = { video: { facingMode: "environment" } }
      }
      navigator.mediaDevices.getUserMedia(props) // { facingMode: { ideal: "environment" } }
        .then(mediaStream => {
          this.cameraStart = true
          this.$refs.videoplay.srcObject = mediaStream
          this.track = mediaStream.getVideoTracks()[0]
          this.imageCapture = new ImageCapture(this.track)
        })
    },
    takePic() { 
      this.imageCapture.takePhoto()
        .then(data => { // on success 
          // this.expanded = true
          createImageBitmap(data) 
          const reader = new FileReader()
          reader.readAsDataURL(data)
          reader.onloadend = () => {
            var img = new Image()
            img.src = reader.result
            img.onload = () => {
              this.datosImagen.contentType = 'image/jpeg'
              this.datosImagen.dataURL = 'data:image/jpeg;base64,'

              const scaleFactor = 1
              this.datosImagen.width = img.width * scaleFactor
              this.datosImagen.height = img.height * scaleFactor
              const elem = document.createElement('canvas')
              elem.width = this.datosImagen.width
              elem.height = this.datosImagen.height
              const ctx = elem.getContext('2d')
              // img.width and img.height will contain the original dimensions
              if (this.datosImagen.width >= this.datosImagen.height) { // rotate
                var TO_RADIANS = Math.PI/180
                elem.width = this.datosImagen.height
                elem.height = this.datosImagen.width
                ctx.translate(elem.width/2 , elem.height/2)
                ctx.rotate( 90 * TO_RADIANS )
                ctx.drawImage(img, -this.datosImagen.width/2, -this.datosImagen.height/2) // 0, 0, width, height)
              } else {
                ctx.drawImage(img, 0, 0)
              }
              var data1 = ctx.canvas.toDataURL(this.datosImagen.contentType, 0.2)
              data1 = data1.substring(this.datosImagen.dataURL.length)
              var raw = data1 // atob(this.datosImagen.dataB64)
              var rawLength = raw.length;
              var uInt8Array = new Uint8Array(rawLength)
              for (var i = 0; i < rawLength; ++i) {
                  uInt8Array[i] = raw.charCodeAt(i)
              }
              this.datosImagen.oMyBlob = new Blob([uInt8Array], {type: this.datosImagen.contentType})
              this.datosImagen.dataB64 = raw

              this.cargarImagen()
              this.$q.dialog({ title: 'Atención', message: '¿ Escanear otra página ?', ok: { label: 'SI', push: true }, cancel: { label: 'NO', color: 'negative' }, persistent: true })
                .onOk(() => {
                  this.doc.addPage()
                  this.takePic()
                })
                .onCancel(() => {
                  this.track.stop()
                  this.cameraStart = false
                  this.expanded = true
                  // this.sendMail()
                })
            }
          }
        })
    },
    cargarImagen() {
      // convierto a PDF
      // var doc = new jsPDF(). 8.3.22
      var imgData = this.datosImagen.dataURL+this.datosImagen.dataB64
      var ratio = this.datosImagen.height / this.datosImagen.width
      if (this.datosImagen.width > this.datosImagen.height) ratio = this.datosImagen.width / this.datosImagen.height
      var w = this.doc.internal.pageSize.getWidth()
      var h = w * ratio
      var dif = h - this.doc.internal.pageSize.getHeight()
      if (dif > 0) {
        w = w - dif
        h = w * ratio
      }
      this.doc.addImage(imgData, 'JPEG', 10, 10,Math.round(w)-20,Math.round(h)-20, '', 'MEDIUM') // this.datosImagen.width, this.datosImagen.height)
    },
    sendMail(){
      this.expanded = false
      var nombre = `digiticket-${date.formatDate(new Date(),'YYYY-MM-DD HHmmss')}`
      var dataPDF = this.doc.output('dataurlstring')  // data:application/pdf;filename=generated.pdf;base64,
      var pos = dataPDF.indexOf(';base64,')+(';base64,').length
      dataPDF = 'data:application/pdf;base64,' + dataPDF.substring(pos)
      this.doc = null
      var vthis = this

      // window.plugins.socialsharing.shareWithOptions(options, onSuccess, onError);
      var file1 = null
      fetch(dataPDF)
      .then(data=>{
        data.blob()
        .then(data1=> {
          console.log(navigator.canShare())
          file1 = new File([data1], (this.datosImagen.subject.length===0?nombre:this.datosImagen.subject)+'.pdf', { type: data1.type }) //(this.datosImagen.subject.length===0?nombre:this.datosImagen.subject)
          console.log(file1)
          navigator.share({
            files: [file1],
            title: `${this.datosImagen.subject}\n\r\n\rPowered by DigiTicket. 2024`,
            text: `${this.datosImagen.body}\n\r\n\rPowered by DigiTicket. 2024`,
          })
          .then(data => {
            vthis.$q.dialog({ title: 'OK', message: 'Ticket subido correctamente ' })
            .onDismiss(()=>{
                  vthis.expanded = false
                })
          })
          .catch(error => {
            vthis.$q.dialog({ title: 'Error', message: 'Error al subir ticket.'+error.message+'. Vuélvalo a intentar o contacte con el administrador' })
          })
        })
      })
      
    }
  }
}
</script>
