<template>
  <b-row>
    <b-col lg="5" class="mb-4"
      ><Control
        @generate="generate"
        @download="download"
        :downloadDisabled="state !== State.FINISH"
      />
    </b-col>
    <b-col lg="7"
      ><Preview :state="state" :loadingPrompt="loadingPrompt" :allCanvas="generatedCanvas"
    /></b-col>
  </b-row>
</template>

<script lang="ts">
import Vue from 'vue';
import jsPDF from 'jspdf';
import Control from './Control.vue';
import Preview from './Preview.vue';
import State from '../utils/state';
import photoLinePaper from '../../public/papers/photos/line';
import paperSize from '../../public/papers/size';
import randomString from '../utils/random';
import createSVG from '../utils/createSVG';
import '../utils/wrapper';

const FACTOR = 10;

export default Vue.extend({
  name: 'HomeBody',
  components: { Control, Preview },
  data() {
    return {
      state: State.BEGIN,
      loadingPrompt: '',
      State,
      generatedCanvas: [] as HTMLCanvasElement[],
      pageConfigForDownload: {
        size: '',
        direction: '',
      },
    };
  },
  methods: {
    customAlert(message: string): void {
      this.$bvToast.toast(message, {
        title: 'Error',
        variant: 'warning',
        solid: true,
        autoHideDelay: 1500,
      });
    },
    async generate(originalForm: formType) {
      const form = originalForm;

      if (form.character.font === 'upload' && form.character.upload === null) {
        this.customAlert(this.$t('alert.noFontUploaded') as string);
        return;
      }

      if (form.paper.source === 'upload' && form.paper.upload === null) {
        this.customAlert(this.$t('alert.noPaperUploaded') as string);
        return;
      }

      if (form.paper.source === 'photo' && form.paper.hasLine === true) {
        form.paper.direction = (photoLinePaper as photoLinePaperType)[form.paper.paper].direction;
        form.paper.lineCount = (photoLinePaper as photoLinePaperType)[form.paper.paper].lineCount;
        form.paper.margin = (photoLinePaper as photoLinePaperType)[form.paper.paper].margin;
      } else {
        const splitMargin = form.paper.marginInText.split(' ');
        if (splitMargin.length !== 4) {
          this.customAlert(this.$t('alert.wrongMarginFormat') as string);
          return;
        }
        form.paper.margin = {
          top: +splitMargin[0],
          right: +splitMargin[1],
          bottom: +splitMargin[2],
          left: +splitMargin[3],
        };
      }

      this.state = State.LOADING;
      this.generatedCanvas = [];
      const revokeList = [];
      this.pageConfigForDownload = {
        size: form.paper.size,
        direction: form.paper.direction,
      };
      const lineWidth = 100 - form.paper.margin.left - form.paper.margin.right;
      const lineHeight = (100 - form.paper.margin.top - form.paper.margin.bottom)
                        / form.paper.lineCount;

      let fontRep = '';
      if (form.text !== '') {
        this.loadingPrompt = this.$t('prompt.loadingFont') as string;
        let fontURL;
        if (form.character.font === 'upload') {
          fontURL = URL.createObjectURL(form.character.upload);
          revokeList.push(fontURL);
        } else {
          fontURL = `./fonts/${form.character.font}`;
        }
        const fontName = randomString(6);

        fontRep = `${Math.ceil(
          ((paperSize as paperSizeType)[form.paper.size].defaultFontSize * form.character.scale)
            / 100,
        )}px "${fontName}"`;

        try {
          const fontLoaded = await new FontFace(fontName, `url("${fontURL}")`).load();
          (document as unknown as { fonts: any }).fonts.add(fontLoaded);
        } catch (err) {
          this.state = State.BEGIN;
          this.customAlert(this.$t('alert.notAFontFile') as string);
          return;
        }
        if (!(document as unknown as { fonts: any }).fonts.check(fontRep)) {
          this.state = State.BEGIN;
          this.customAlert(this.$t('alert.fontNotSupported') as string);
          return;
        }
      }

      this.loadingPrompt = this.$t('prompt.loadingPaper') as string;
      let imgLoaded: HTMLImageElement;
      try {
        imgLoaded = await new Promise((resolve, reject) => {
          const img = new Image();
          img.onload = () => {
            resolve(img);
          };
          img.onerror = reject;
          if (form.paper.source === 'photo') {
            img.src = `./papers/photos/${form.paper.hasLine ? 'line' : 'noline'}/${
              form.paper.paper
            }`;
          } else if (form.paper.source === 'origin') {
            const svg = createSVG(
              form.paper.direction === 'vertical'
                ? (paperSize as paperSizeType)[form.paper.size].width
                : (paperSize as paperSizeType)[form.paper.size].height,
              form.paper.direction === 'vertical'
                ? (paperSize as paperSizeType)[form.paper.size].height
                : (paperSize as paperSizeType)[form.paper.size].width,
              form.paper.lineCount,
              form.paper.margin,
              form.paper.backgroundColor,
              form.paper.hasLine,
            );
            const blob = new Blob([svg], { type: 'image/svg+xml' });
            const imgURL = URL.createObjectURL(blob);
            revokeList.push(imgURL);
            img.src = imgURL;
          } else if (form.paper.source === 'upload') {
            const imgURL = URL.createObjectURL(form.paper.upload);
            revokeList.push(imgURL);
            img.src = imgURL;
          }
        });
      } catch (err) {
        this.state = State.BEGIN;
        this.customAlert(this.$t('alert.wrongPaperFile') as string);
        return;
      }

      this.loadingPrompt = this.$t('prompt.typesetting') as string;
      const tempCanvas = document.createElement('canvas');
      const tempCtx = tempCanvas.getContext('2d')!;

      tempCanvas.width = form.paper.direction === 'vertical'
        ? (paperSize as paperSizeType)[form.paper.size].width * FACTOR
        : (paperSize as paperSizeType)[form.paper.size].height * FACTOR;
      tempCanvas.height = form.paper.direction === 'vertical'
        ? (paperSize as paperSizeType)[form.paper.size].height * FACTOR
        : (paperSize as paperSizeType)[form.paper.size].width * FACTOR;

      tempCtx.font = fontRep;
      const textLines = tempCtx.typesetText(
        // Draw canvas even with empty text,
        // in order to preview paper with empty text
        form.text === '' ? ' ' : form.text,
        form.character.spacing,
        lineWidth,
      );

      for (let i = 0, j = 0; i < textLines.length; i += form.paper.lineCount, j += 1) {
        this.loadingPrompt = this.$t('prompt.generating', { count: j + 1 }) as string;

        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d')!;
        canvas.width = form.paper.direction === 'vertical'
          ? (paperSize as paperSizeType)[form.paper.size].width * FACTOR
          : (paperSize as paperSizeType)[form.paper.size].height * FACTOR;
        canvas.height = form.paper.direction === 'vertical'
          ? (paperSize as paperSizeType)[form.paper.size].height * FACTOR
          : (paperSize as paperSizeType)[form.paper.size].width * FACTOR;
        ctx.translate(canvas.width / 2, canvas.height / 2);
        const radian = ((Math.random() - 0.5) * 2 * form.simulation.paperRotation * Math.PI) / 180;
        ctx.rotate(radian);
        const radianAbs = Math.abs(radian);
        let k = canvas.height / canvas.width;
        if (k > 1) {
          k = 1 / k;
        }
        const scaleFactor = (Math.tan(radianAbs) / k + 1) * Math.cos(radianAbs);
        ctx.scale(scaleFactor, scaleFactor);
        ctx.translate(-canvas.width / 2, -canvas.height / 2);

        if (!form.paper.hasLine && form.paper.direction === 'horizontal') {
          ctx.translate(canvas.width / 2, canvas.height / 2);
          ctx.rotate((90 * Math.PI) / 180);
          ctx.drawImage(
            imgLoaded,
            -canvas.height / 2,
            -canvas.width / 2,
            canvas.height,
            canvas.width,
          );
          ctx.rotate((-90 * Math.PI) / 180);
          ctx.translate(-canvas.width / 2, -canvas.height / 2);
        } else {
          ctx.drawImage(imgLoaded, 0, 0, canvas.width, canvas.height);
        }

        ctx.font = fontRep;
        ctx.fillStyle = form.character.color;
        ctx.textBaseline = 'bottom';
        const { textEffect } = form.simulation;
        ctx.filter = `blur(${textEffect.blurRadius}px) opacity(${textEffect.opacity}%) drop-shadow(${textEffect.shadow.offset.horizontal}px ${textEffect.shadow.offset.vertical}px ${textEffect.shadow.radius}px ${textEffect.shadow.color})`;
        // eslint-disable-next-line no-await-in-loop
        await new Promise((resolve) => {
          setTimeout(() => {
            textLines.slice(i, i + form.paper.lineCount).forEach((lineString, index) => {
              ctx.fillTextExtended(
                lineString,
                ((form.paper.margin.left
                  + (Math.random() - 0.5) * 2 * form.simulation.randomOffset.lineBeginning)
                  / 100)
                  * canvas.width,
                ((form.paper.margin.top + lineHeight * (index + 1)) / 100) * canvas.height,
                form.character.spacing,
                form.simulation.textEffect.distortion,
                form.simulation.randomOffset.horizontal,
                form.simulation.randomOffset.vertical,
              );
            });
            resolve();
          }, 0);
        });
        this.generatedCanvas.push(canvas);
      }

      revokeList.forEach((e) => {
        URL.revokeObjectURL(e);
      });
      this.state = State.FINISH;
    },

    download(): void {
      // eslint-disable-next-line new-cap
      const pdf = new jsPDF({
        format: this.pageConfigForDownload.size,
        orientation: this.pageConfigForDownload.direction === 'vertical' ? 'portrait' : 'landscape',
      });

      this.generatedCanvas.forEach((canvas, index) => {
        if (index !== 0) {
          pdf.addPage();
        }
        pdf.addImage(
          canvas,
          0,
          0,
          this.pageConfigForDownload.direction === 'vertical'
            ? (paperSize as paperSizeType)[this.pageConfigForDownload.size].width
            : (paperSize as paperSizeType)[this.pageConfigForDownload.size].height,
          this.pageConfigForDownload.direction === 'vertical'
            ? (paperSize as paperSizeType)[this.pageConfigForDownload.size].height
            : (paperSize as paperSizeType)[this.pageConfigForDownload.size].width,
        );
      });
      pdf.save(`download-${new Date().toLocaleString().replace(/[^a-zA-Z0-9]+/g, '-')}.pdf`);
    },
  },
});
</script>
