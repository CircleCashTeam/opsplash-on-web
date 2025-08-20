<template>
  <div style="position: relative; overflow: hidden;">
    <mdui-top-app-bar scroll-behavior="elevate">
      <!--mdui-button-icon icon="menu"></mdui-button-icon-->
      <mdui-icon style="font-size: 36px">
        <svg xmlns="http://www.w3.org/2000/svg" width="100" height="100" viewBox="0 0 100 100">
          <!-- 红色粗圆 -->
          <!--circle cx="50" cy="50" r="40" stroke="red" stroke-width="8" fill="none" /-->
          <!-- 货币符号 -->
          <text x="50" y="80" font-family="Arial" font-size="80" text-anchor="middle">⭕</text>
          <text x="50" y="65" font-family="Arial" font-size="60" text-anchor="middle">💰</text>
        </svg>
      </mdui-icon>
      <mdui-top-app-bar-title>OPSPLASH on Web</mdui-top-app-bar-title>
      <div style="flex-grow: 1"></div>
      <mdui-tooltip content="About">
        <mdui-button-icon icon="info--outlined" @click="showInfo"></mdui-button-icon>
      </mdui-tooltip>
    </mdui-top-app-bar>

    <h2 style="margin-left: 1em;">Actions</h2>
    <mdui-card variant="outlined"
      style="display: flex ;padding: 10px; width: 100%; max-width: none !important; flex-wrap: wrap; gap: 20px;">
      <mdui-button style="flex-grow: 1; max-width: 20em;" variant="elevated"
        @click="this.$refs.inputFile.click">Upload</mdui-button>
      <input style="display: none;" type="file" name="input" id="input" ref="inputFile" @change="handleChangeFile">
      <mdui-button style="flex-grow: 1; max-width: 20em;" variant="elevated" @click="testSplashImage"
        ref="testBtn">Test</mdui-button>
    </mdui-card>

    <div style="display: flex; align-items: center;">
      <h2 style="margin-left: 1em;">Result</h2>
      <div style="flex-grow: 1;"></div>
      <mdui-button @click="this.term.clear" variant="outlined">Clear</mdui-button>
    </div>
    <mdui-card variant="filled"
      style="width: 100% ;flex-grow: 1; padding: 10px; padding-bottom: 0; padding-right: 0; margin-bottom: 1em; background-color: #1f1f1f;">
      <div id="terminal" ref="terminal" style="width: 100%; flex-grow: 1;height: 16rem;"></div>
    </mdui-card>

    <mdui-divider></mdui-divider>
    <h2 style="margin-left: 1em;">Images</h2>
    <mdui-list ref="listview">
    </mdui-list>
    <mdui-tooltip content="Download">
      <mdui-fab style="position: fixed; right: 20px; bottom: 20px; z-index: 1000;" icon="download"
        :disabled="isDownloadDisabled" @click="download"></mdui-fab>
    </mdui-tooltip>

    <mdui-dialog ref="imagePreviewDialog" close-on-overlay-click close-on-esc>
      <span slot="headline">Preview image</span>
      <div slot="description" style="overflow-y: hidden;">
        <img :src="imgSrc" style="height: 100%; width: 100%; object-fit: contain; overflow-y: hidden;" :alt="imgName"
          ref="img">
      </div>
      <mdui-button slot="action" variant="text" @click="downloadImg">Save</mdui-button>
      <mdui-button slot="action" variant="tonal" @click="this.$refs.changeInput.click">Replace</mdui-button>
      <input style="display: none;" ref="changeInput" type="file" name="replace" id="replace" accept="image/*"
        @change="onReplaceChange">
    </mdui-dialog>
  </div>
</template>

<style>
.xterm-viewport {
  overflow: hidden !important;
}
</style>

<script>
import { dialog } from 'mdui/functions/dialog.js';
import { Terminal } from '@xterm/xterm';
import { FitAddon } from 'xterm-addon-fit';
import { OPPOSPlashImage } from './utils/opsplash.mts';
import 'mdui/components/list-item.js';
import 'mdui/components/list-subheader.js';
import { ListItem } from 'mdui/components/list-item.js';
import { Button } from 'mdui';

export default {
  name: "AppBar",
  data() {
    return {
      term: null,
      isDownloadDisabled: true,
      uploadFile: null,
      splashImage: null,
      imgSrc: '',
      imgName: '',
      imgIdx: 0,
    }
  },
  mounted() {
    this.term = new Terminal({
      allowProposedApi: true,
      fontFamily: "'Roboto Mono', monospace",
      theme: {
        background: '#1f1f1f',
      },
    })
    const fitAddon = new FitAddon()

    this.term.open(this.$refs.terminal)
    this.term.loadAddon(fitAddon)
    this.term.write("Hello world!\n\r")

    window.addEventListener('resize', () => {
      fitAddon.fit();
    })

    this.$nextTick(() => {
      fitAddon.fit()
    })

    // replace console.log into terminal
    const origLog = console.log

    console.log = (...args) => {
      this.term.write(`${args.join(' ')}\n\r`)
      origLog.apply(console, args)
    }

    console.log("Test log replace...")
  },
  methods: {
    download() {
      const blob = this.splashImage.genNewImage()

      const a = document.createElement('a')
      const url = URL.createObjectURL(blob)

      a.href = url
      a.download = "new-splash.img"

      a.click()

      document.removeChild(a)
      URL.revokeObjectURL(url)
    },
    testSplashImage() {
      const testButton = this.$refs.testBtn;
      if (this.splashImage) {
        testButton.loading = true;
        this.splashImage.test().then(() => {
          testButton.loading = false;
        });
      } else {
        console.log("Please upload splash image first...")
      }
    },
    handleChangeFile(event) {
      this.uploadFile = event.target.files[0]

      if (this.uploadFile) {
        this.term.write(`File upload: ${this.uploadFile.name}\n\rSize: ${this.uploadFile.size}\n\r`)
        const reader = new FileReader()
        reader.onload = (e) => {
          const result = e.target.result

          try {
            this.splashImage = new OPPOSPlashImage(result, true)
            this.isDownloadDisabled = false
            this.updateList()
          } catch (error) {
            this.term.write(`${error}\n\r`)
          }
        }

        reader.onerror = (e) => {
          this.term.write(`File read failed\n\r${e}\n\r`)
        }

        reader.readAsArrayBuffer(this.uploadFile)
      }
    },
    showInfo() {
      dialog({
        headline: "About opsplash on web",
        description: "opsplash is a tool to modified OPPO/Realme/Oneplus splash screen when boot up.\nIf you edit done, press download button to download splash image back.",
        actions: [
          {
            text: "Github",
            onClick: () => {
              window.open("https://github.com/CircleCashTeam", '_blank')
              return false
            }
          },
          {
            text: "OK",
            onClick: () => {
              console.log("confirmed");
              return true;
            },
          }
        ]
      });
    },
    updateList() {
      const listview = this.$refs.listview;
      // Remove expired item
      while (listview.firstChild) {
        listview.removeChild(listview.firstChild);
      }

      this.splashImage.dataInfos.forEach((dataInfo, index) => {
        const item = new ListItem();

        item.headline = `${dataInfo.name}`
        item.description = `offset: ${dataInfo.offset}\nreal size: ${dataInfo.realsz}\ncompressed size: ${dataInfo.compsz}`
        item.icon = 'image--outlined'

        item.addEventListener('click', () => {
          console.log("Item clicked!", index);
          const blob = this.splashImage.getRawImageBlobByIndex(index);
          const url = URL.createObjectURL(blob);
          this.imgSrc = url;
          this.imgName = dataInfo.name;
          this.imgIdx = index;
          this.$refs.imagePreviewDialog.open = true;
        });

        listview.appendChild(item);
      });
    },
    downloadImg() {
      const a = document.createElement('a')
      a.href = this.imgSrc
      a.download = this.imgName

      console.log(this.imgName)
      a.click()
    },
    onReplaceChange(event) {
      const img = this.$refs.img
      const file = event.target.files[0]
      if (file) {
        const reader = new FileReader()
        let noReplace = false
        // Close dialog
        this.$refs.imagePreviewDialog.open = false

        let oWidth = img.naturalWidth
        let oHeight = img.naturalHeight
        console.log("Get original width:", oWidth, "height:", oHeight)

        reader.onload = (e) => {
          const image = new Image()

          image.onload = () => {
            const canvas = document.createElement('canvas')
            canvas.width = image.width
            canvas.height = image.height

            if (image.width != oWidth && image.height != oHeight) {
              dialog({
                headline: "Detect different size with original image",
                description: "Replace different size image may cause image show small than screen, are you sure?",
                icon: "warning",
                actions: [
                  {
                    text: "No",
                    onClick: () => {
                      noReplace = true
                      return true
                    }
                  },
                  {
                    text: "Yes",
                    onClick: () => {
                      return true
                    }
                  },
                ]
              })
            }

            if (!noReplace) {
              const ctx = canvas.getContext('2d')
              ctx.drawImage(image, 0, 0)

              const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
              this.splashImage.changeImageByIndex(this.imgIdx, imageData)
            } else {
              console.log("Image not replace...")
            }
          }

          image.src = e.target.result
        }

        reader.readAsDataURL(file)
      }
    }
  }
}

</script>