<template>
  <div class="container">
    <h1>LaTeX to SVG Converter</h1>

    <div class="input-section">
      <textarea v-model="latexInput" placeholder="Enter LaTeX expression (e.g. $\int_{a}^{b} f(x) \, dx$)"
        @input="renderLatex"></textarea>

      <div class="options">
        <div class="color-options">
          <label>
            Background:
            <input type="color" v-model="bgColor" @change="renderLatex">
          </label>
          <label>
            <input type="checkbox" v-model="transparentBg" @change="renderLatex"> Transparent
          </label>
          
          <label>
            Text Color:
            <input type="color" v-model="textColor" @change="renderLatex">
          </label>
        </div>
      </div>
    </div>

    <div class="preview-section">
      <h2>Preview:</h2>
      <div ref="previewContainer" class="preview-container"></div>
    </div>

    <div class="download-section">
      <button @click="downloadSVG" :disabled="!svgContent">Download SVG</button>
      <div v-if="error" class="error">{{ error }}</div>
    </div>
  </div>
</template>

<script lang="ts">
declare global {
  interface Window {
    MathJax: any;
  }
}

export default {
  name: 'App',

  data() {
    return {
      latexInput: "f_n = \\frac{n v}{2L}, \\quad v = \\sqrt{\\frac{T}{\\mu}}",
      svgContent: '',
      error: '',
      bgColor: '#ffffff',
      textColor: '#ffffff',
      transparentBg: true
    };
  },

  mounted() {
    this.loadMathJax().then(() => {
      this.renderLatex();
    });
  },

  methods: {
    loadMathJax(): Promise<void> {
      return new Promise((resolve) => {
        if (window.MathJax) {
          resolve();
          return;
        }

        const script = document.createElement('script');
        script.src = 'https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-svg.js';
        script.async = true;
        script.onload = () => {
          window.MathJax.startup.promise.then(() => {
            resolve();
          });
        };
        document.head.appendChild(script);
      });
    },

    async renderLatex() {
      const container = this.$refs.previewContainer as HTMLElement;
      if (!container) return;

      this.error = '';

      try {
        await this.loadMathJax();

        container.innerHTML = '';
        const output = await window.MathJax.tex2svg(this.latexInput, { display: true });

        const svgElement = output.querySelector('svg');
        if (svgElement) {
          svgElement.style.transformOrigin = 'top left';

          // Apply text color to all relevant elements in the preview
          this.applyTextColor(svgElement, this.textColor);
        }

        container.appendChild(output);

        // Extract SVG content for download
        if (svgElement) {
          // Clone the SVG to modify it for download
          const svgClone = svgElement.cloneNode(true) as SVGElement;

          // Add a CSS style element inside the SVG for color control
          const styleElement = document.createElementNS("http://www.w3.org/2000/svg", "style");
          styleElement.textContent = `
            * { color: ${this.textColor}; }
            text, path, line, rect, use, g[data-mml-node] * { 
              fill: ${this.textColor} !important; 
              stroke: ${this.textColor} !important;
            }
            [fill="none"] { fill: none !important; }
          `;
          svgClone.insertBefore(styleElement, svgClone.firstChild);

          // Set background if not transparent
          if (!this.transparentBg) {
            svgClone.style.backgroundColor = this.bgColor;
            svgClone.setAttribute('style', `background-color: ${this.bgColor};`);
          }

          this.svgContent = svgClone.outerHTML;
        }
      } catch (e) {
        this.error = `Error rendering LaTeX: ${e instanceof Error ? e.message : String(e)}`;
        this.svgContent = '';
      }
    },

    applyTextColor(svgElement: SVGElement, color: string) {
      // Apply to paths
      const paths = svgElement.querySelectorAll('path');
      paths.forEach((path: SVGPathElement) => {
        path.setAttribute('fill', color);
        path.setAttribute('stroke', color);
      });

      // Apply to text elements
      const texts = svgElement.querySelectorAll('text');
      texts.forEach((text: SVGTextElement) => {
        text.setAttribute('fill', color);
      });

      // Apply to use elements
      const uses = svgElement.querySelectorAll('use');
      uses.forEach((use: SVGUseElement) => {
        use.setAttribute('fill', color);
      });

      // Apply to rect elements that are part of text
      const rects = svgElement.querySelectorAll('rect[data-mml-node]');
      rects.forEach((rect) => {
        const svgRect = rect as SVGRectElement;
        svgRect.setAttribute('fill', color);
      });

      // Set stroke color for elements with stroke
      const strokeElements = svgElement.querySelectorAll('[stroke="currentColor"]');
      strokeElements.forEach((el) => {
        (el as SVGElement).setAttribute('stroke', color);
      });

      // Set stroke color for all line elements (horizontal lines in fractions, etc.)
      const lines = svgElement.querySelectorAll('line');
      lines.forEach((line: SVGLineElement) => {
        line.setAttribute('stroke', color);
      });

      // Apply color to all elements with data-mml-node attribute (MathML nodes)
      const mathNodes = svgElement.querySelectorAll('[data-mml-node]');
      mathNodes.forEach((node: Element) => {
        if (node.hasAttribute('stroke')) {
          node.setAttribute('stroke', color);
        }
        if (node.hasAttribute('fill') && node.getAttribute('fill') !== 'none') {
          node.setAttribute('fill', color);
        }
      });
    },

    downloadSVG() {
      if (!this.svgContent) return;

      const blob = new Blob([this.svgContent], { type: 'image/svg+xml' });
      const url = URL.createObjectURL(blob);

      const a = document.createElement('a');
      a.href = url;
      a.download = 'latex-equation.svg';
      document.body.appendChild(a);
      a.click();

      setTimeout(() => {
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      }, 100);
    }
  }
};
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
}

h1 {
  text-align: center;
  margin-bottom: 30px;
  color: #dedede;
}

.input-section {
  margin-bottom: 20px;
}

textarea {
  width: 100%;
  height: 100px;
  padding: 10px;
  font-family: monospace;
  border: 1px solid #ddd;
  border-radius: 4px;
  resize: vertical;
}

.options {
  margin-top: 10px;
  display: flex;
  flex-direction: column;
  gap: 15px;
}

label {
  background-color: #343434;
  padding: 10px;
  font-weight: bold;
  flex: auto;
  justify-content: space-between;
  border-radius: 1rem;
}

.color-options {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  align-items: center;
}

.preview-section {
  margin-bottom: 20px;
  padding: 15px;
  border: 1px solid #eee;
  border-radius: 4px;
}

.preview-container {
  min-height: 100px;
  display: flex;
  justify-content: center;
  padding: 15px 0;
  overflow-x: auto;
}

.download-section {
  text-align: center;
}

button {
  padding: 10px 20px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: #45a049;
}

.error {
  color: red;
  margin-top: 10px;
}
</style>