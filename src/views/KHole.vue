<template>
  <div class="khole">
    <canvas ref="glCanvas" class="gl-canvas"></canvas>
    <div class="content">
      <p :class="['contract-address', { 'fade-in': showContract }]">
        {{ contractAddress }}
      </p>
      <h1 :class="['welcome-text', { 'fade-in': showWelcome }]">
        Welcome to the K-Hole
      </h1>
      <img :src="require('@/assets/hazzyCat.png')" alt="cat-in-hazmat-suit"
           :class="['ket-image', { 'fade-in': showKet }]">
      <div :class="['icons', { 'fade-in': showIcons }]">
        <a href="https://t.me/" target="_blank">
          <img src="https://cdnjs.cloudflare.com/ajax/libs/simple-icons/9.21.0/telegram.svg" alt="Telegram">
        </a>
        <a href="https://x.com/" target="_blank">
          <img src="https://cdnjs.cloudflare.com/ajax/libs/simple-icons/9.21.0/x.svg" alt="X (Twitter)">
        </a>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'KHole',
  data() {
    return {
      contractAddress: 'Are you ready?',
      showContract: false,
      showWelcome: false,
      showKet: false,
      showIcons: false,
      gl: null,
      program: null,
      positionBuffer: null,
      positionAttributeLocation: null,
      resolutionUniformLocation: null,
      timeUniformLocation: null,
      startTime: null
    }
  },
  mounted() {
    this.initWebGL();
    this.setFavicon();
    this.animateContent();
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.resizeCanvas);
    cancelAnimationFrame(this.animationFrame);
  },
  methods: {
    initWebGL() {
      const canvas = this.$refs.glCanvas;
      this.gl = canvas.getContext('webgl');

      if (!this.gl) {
        console.error('Unable to initialize WebGL. Your browser may not support it.');
        return;
      }

      const vertexShaderSource = `
        attribute vec4 aVertexPosition;
        void main() {
          gl_Position = aVertexPosition;
        }
      `;

      const fragmentShaderSource = `
        precision highp float;
        uniform vec2 iResolution;
        uniform float iTime;

        vec3 hash(vec3 p){
            p = vec3( dot(p,vec3(127.1,311.7, 74.7)),
                      dot(p,vec3(269.5,183.3,246.1)),
                      dot(p,vec3(113.5,271.9,124.6)));
            return -1.0 + 2.0*fract(sin(p)*43758.5453123);
        }

        float noise_3d(in vec3 p){
            vec3 i = floor( p );
            vec3 f = fract( p );

            vec3 u = f*f*(3.0-2.0*f);
            return mix( mix( mix( dot( hash( i + vec3(0.0,0.0,0.0) ), f - vec3(0.0,0.0,0.0) ),
                                  dot( hash( i + vec3(1.0,0.0,0.0) ), f - vec3(1.0,0.0,0.0) ), u.x),
                             mix( dot( hash( i + vec3(0.0,1.0,0.0) ), f - vec3(0.0,1.0,0.0) ),
                                  dot( hash( i + vec3(1.0,1.0,0.0) ), f - vec3(1.0,1.0,0.0) ), u.x), u.y),
                        mix( mix( dot( hash( i + vec3(0.0,0.0,1.0) ), f - vec3(0.0,0.0,1.0) ),
                                  dot( hash( i + vec3(1.0,0.0,1.0) ), f - vec3(1.0,0.0,1.0) ), u.x),
                             mix( dot( hash( i + vec3(0.0,1.0,1.0) ), f - vec3(0.0,1.0,1.0) ),
                                  dot( hash( i + vec3(1.0,1.0,1.0) ), f - vec3(1.0,1.0,1.0) ), u.x), u.y), u.z );
        }

        float fbm (vec2 uv, float amplitude, float lacunarity, float gain) {
            float value = 0.0;
            float frequency = 1.0;

            for (int i = 0; i < 10; i++) {
                value += amplitude * abs(noise_3d(vec3(uv * frequency, iTime * 0.5)) * 7.0);
                frequency *= lacunarity;
                amplitude *= gain;
            }

            return value;
        }

        vec2 kaleidoscope(vec2 uv) {
            vec2 centered = uv - 0.5;
            float angle = atan(centered.y, centered.x);
            float radius = length(centered);

            float sector = floor(angle / (3.14159 / 3.0)) + 3.0;
            angle = mod(angle, 3.14159 / 3.0) - 3.14159 / 6.0;

            vec2 reflected = vec2(cos(angle), sin(angle)) * radius;
            reflected = abs(reflected);

            return reflected;
        }

        vec2 rotate2D(vec2 uv, float angle) {
            uv -= 0.5;
            uv = mat2(cos(angle), -sin(angle), sin(angle), cos(angle)) * uv;
            uv += 0.5;
            return uv;
        }

        void main() {
            vec2 fragCoord = gl_FragCoord.xy;
            vec2 uv = fragCoord/iResolution.xy;

            float aspect = iResolution.x / iResolution.y;
            uv.x *= aspect;

            uv = uv - vec2(aspect * 0.5, 0.5);
            uv = rotate2D(uv + vec2(0.5), iTime * 0.1);
            uv = kaleidoscope(uv);
            uv = rotate2D(uv, iTime * -0.05);

            float transitionValue = min(3.0, iTime * 0.1);

            float n = fbm(uv * max(0.01, transitionValue), 0.5, sin(iTime * 0.1) * 1.0 + 5.0, abs(sin(iTime * 0.2)) * 0.5 + 0.1);
            n = sin(iTime) * 0.4 + 1.2 - n;
            n = n * n * n * n;

            vec2 color_uv = uv;
            color_uv *= 2.5;
            vec3 color = mix(vec3(0.9, 0.0, 0.1), vec3(0.6, sin(iTime) * 0.2 + 0.5, 0.8), length(color_uv));
            gl_FragColor = vec4(mix(color, vec3(0.0), 1.0 - n), 1.0);
        }
      `;

      const vertexShader = this.createShader(this.gl.VERTEX_SHADER, vertexShaderSource);
      const fragmentShader = this.createShader(this.gl.FRAGMENT_SHADER, fragmentShaderSource);

      this.program = this.createProgram(vertexShader, fragmentShader);

      this.positionAttributeLocation = this.gl.getAttribLocation(this.program, 'aVertexPosition');
      this.resolutionUniformLocation = this.gl.getUniformLocation(this.program, 'iResolution');
      this.timeUniformLocation = this.gl.getUniformLocation(this.program, 'iTime');

      this.positionBuffer = this.gl.createBuffer();
      this.gl.bindBuffer(this.gl.ARRAY_BUFFER, this.positionBuffer);

      const positions = [
        -1, -1,
         1, -1,
        -1,  1,
         1,  1,
      ];
      this.gl.bufferData(this.gl.ARRAY_BUFFER, new Float32Array(positions), this.gl.STATIC_DRAW);

      this.startTime = Date.now();
      window.addEventListener('resize', this.resizeCanvas);
      this.resizeCanvas();
      this.render();
    },
    createShader(type, source) {
      const shader = this.gl.createShader(type);
      this.gl.shaderSource(shader, source);
      this.gl.compileShader(shader);
      if (!this.gl.getShaderParameter(shader, this.gl.COMPILE_STATUS)) {
        console.error('An error occurred compiling the shaders: ' + this.gl.getShaderInfoLog(shader));
        this.gl.deleteShader(shader);
        return null;
      }
      return shader;
    },
    createProgram(vertexShader, fragmentShader) {
      const program = this.gl.createProgram();
      this.gl.attachShader(program, vertexShader);
      this.gl.attachShader(program, fragmentShader);
      this.gl.linkProgram(program);
      if (!this.gl.getProgramParameter(program, this.gl.LINK_STATUS)) {
        console.error('Unable to initialize the shader program: ' + this.gl.getProgramInfoLog(program));
        return null;
      }
      return program;
    },
    resizeCanvas() {
      const canvas = this.gl.canvas;
      const displayWidth = canvas.clientWidth;
      const displayHeight = canvas.clientHeight;
      if (canvas.width !== displayWidth || canvas.height !== displayHeight) {
        canvas.width = displayWidth;
        canvas.height = displayHeight;
        this.gl.viewport(0, 0, canvas.width, canvas.height);
      }
    },
    render() {
      this.resizeCanvas();
      this.gl.clearColor(0.0, 0.0, 0.0, 1.0);
      this.gl.clear(this.gl.COLOR_BUFFER_BIT);

      this.gl.useProgram(this.program);

      this.gl.enableVertexAttribArray(this.positionAttributeLocation);
      this.gl.bindBuffer(this.gl.ARRAY_BUFFER, this.positionBuffer);
      this.gl.vertexAttribPointer(this.positionAttributeLocation, 2, this.gl.FLOAT, false, 0, 0);

      this.gl.uniform2f(this.resolutionUniformLocation, this.gl.canvas.width, this.gl.canvas.height);
      this.gl.uniform1f(this.timeUniformLocation, (Date.now() - this.startTime) / 1000);

      this.gl.drawArrays(this.gl.TRIANGLE_STRIP, 0, 4);

      this.animationFrame = requestAnimationFrame(this.render);
    },
    setFavicon() {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        canvas.width = 32;
        canvas.height = 32;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, 32, 32);
        const faviconUrl = canvas.toDataURL('image/png');
        const link = document.createElement('link');
        link.type = 'image/x-icon';
        link.rel = 'shortcut icon';
        link.href = faviconUrl;
        document.head.appendChild(link);
      };
      img.src = require('@/assets/kCat.png');
    },
    animateContent() {
      setTimeout(() => {
        this.showWelcome = true;
        setTimeout(() => {
        this.showContract = true;
          setTimeout(() => {
            this.showKet = true;
            setTimeout(() => {
              this.showIcons = true;
            }, 1000);
          }, 1000);
        }, 1000);
      }, 500);
    }
  }
}
</script>

<style scoped>
.ket-hole {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  font-family: Arial, sans-serif;
}

.gl-canvas {
  display: block;
  width: 100%;
  height: 100%;
  position: fixed;
  top: 0;
  left: 0;
}

.content {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  color: white;
  z-index: 10;
}

.contract-address {
  font-size: 0.8em;
  margin-bottom: 10px;
  opacity: 0;
  transition: opacity 2s ease-in-out;
  word-break: break-all;
}

.welcome-text {
  font-size: 2em;
  margin-bottom: 20px;
  opacity: 0;
  transition: opacity 2s ease-in-out;
}

.ket-image {
  max-width: 50%;
  max-height: 50%;
  opacity: 0;
  transition: opacity 2s ease-in-out;
}

.icons {
  margin-top: 20px;
  opacity: 0;
  transition: opacity 2s ease-in-out;
}

.icons a {
  display: inline-block;
  margin: 0 10px;
  color: white;
  text-decoration: none;
}

.icons img {
  width: 30px;
  height: 30px;
  filter: invert(1);
}

.fade-in {
  opacity: 1 !important;
}
</style>
