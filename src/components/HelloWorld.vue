<template>
  <div id="WebGL-output" ref="WebGL">

  </div>
</template>

<script>
  import * as THREE from 'three';
  import TrackballControls from 'three-trackballcontrols';
  import OrbitControls from 'three-orbitcontrols';
  import {CSS2DRenderer, CSS2DObject} from 'three-css2drender';
  import Stats from 'stats-js';
  import * as dat from 'dat.gui';
  import * as d3 from 'd3';
  import {scaleOrdinal, schemeCategory20} from 'd3'

  export default {
    name: 'webgl',
    data() {
      return {
        fov: 45,
        near: 1,
        far: 100000,
        gData1: null,
        gData2: null,
        gData3: null,
        forceSate: {},
        flag: true
      }
    },
    mounted() {
      this.camera = null,
        this.scene = null ,
        this.controls = null ,
        this.renderer = null,
        this.stats = null,
        this.raycaster = null,
        this.mouse = null,
        this.InterSelected = null,
        this.light = null;
      this.color = this.getColor();
      this.labelRenderer = null;
      this.init();
      this.computed();
      this.statsInit();
      this.animate();
      this.guiInit();
    },
    methods: {
      init() {
        let self = this;
        this.mouse = new THREE.Vector2();
        this.raycaster = new THREE.Raycaster();

        let width = window.innerWidth;
        let height = window.innerHeight;
        // 创建场景
        this.scene = new THREE.Scene();
        // 初始化相机
        this.camera = new THREE.PerspectiveCamera(this.fov, width / height, this.near, this.far);   //创建摄像机
        this.camera.up = new THREE.Vector3(0, 1, 0); // 设置y轴向上
        this.camera.layers.enable(0);
        this.camera.layers.enable(1);
        this.camera.layers.enable(2);
        this.camera.layers.enable(3);
        this.camera.position.z = 2299;
        this.camera.position.x = -1906;
        this.camera.position.y = -280;
        // this.camera.lookAt({x: 0, y: 0, z: 0});
        this.camera.lookAt(this.scene.position);


        this.light = new THREE.AmbientLight(0xffffff);
        this.light.position.set(1, 1, 1);
        this.light.layers.enable(0);
        this.light.layers.enable(1);
        this.light.layers.enable(2);
        this.light.layers.enable(3);
        this.scene.add(this.light);


        // 渲染器  使用WebGL渲染器  同时还支持canvas svg
        this.renderer = new THREE.WebGLRenderer({
          antialias: true
        });
        this.renderer.setClearColor(new THREE.Color(0x303030));
        this.renderer.setSize(width, height);
        this.renderer.setPixelRatio(window.devicePixelRatio);//为了兼容高清屏幕
        document.getElementById("WebGL-output").appendChild(self.renderer.domElement);

        // label渲染器
        this.labelRenderer = new CSS2DRenderer();
        this.labelRenderer.setSize(window.innerWidth, window.innerHeight);
        this.labelRenderer.domElement.style.position = 'absolute';
        this.labelRenderer.domElement.style.top = 0;
        document.getElementById("WebGL-output").appendChild(this.labelRenderer.domElement);

        // 初始化控制器
        self.controls = new TrackballControls(this.camera, this.$refs.WebGL);
        self.controls.rotateSpeed = 2.5;
        self.controls.zoomSpeed = 1.2;
        self.controls.panSpeed = 0.8;
        self.controls.noZoom = false;
        self.controls.noPan = false;
        self.controls.staticMoving = true;
        self.controls.dynamicDampingFactor = 0.3;

        // 坐标系可视化
        // let axesHelper = new THREE.AxesHelper(1000);
        // self.scene.add(axesHelper);

        // 添加窗口监听事件（resize-onresize即窗口或框架被重新调整大小）
        window.addEventListener('resize', this.onWindowResize, false);
        document.addEventListener('mousemove', this.onDocumentMouseMove, false);
      },
      addObj(data, index) {
        let self = this;
        // 添加圆
        // 几何体  材质(设置颜色 透明度等 transparent要设置为true才能有效果)
        let geometrySphere = new THREE.SphereGeometry(10, 10, 10);
        data.nodes.forEach(item => {
          let circle = new THREE.Mesh(geometrySphere, new THREE.MeshLambertMaterial({
            color: self.color(index),
            opacity: 0.5,
            transparent: true
          }));
          circle.layers.set(index);
          circle.position.set(item.x, item.y, index * 500);
          this.scene.add(circle);
        });

        // 添加线
        // 材质要单独添加 不然交互时 会同时触发
        data.links.forEach(item => {
          let material = new THREE.LineBasicMaterial({
            opacity: 0.5,
            transparent: true,
            vertexColors: true // 支持颜色插值
          });
          let color1 = new THREE.Color(0x0000FF), color2 = new THREE.Color(0xFF0000);
          // 声明一个几何体 给几何体添加位置节点和颜色
          let geometry = new THREE.Geometry();
          geometry.vertices.push(new THREE.Vector3(item.source.x, item.source.y, index * 500));
          geometry.vertices.push(new THREE.Vector3(item.target.x, item.target.y, index * 500));
          geometry.colors.push(color1, color2);
          let line = new THREE.Line(geometry, material);
          line.layers.set(index);
          this.scene.add(line);
        });

        // 添加平面
        // 材质要单独添加 不然交互时 会同时触发
        //  常见几何材料
        //  MeshBasicMaterial：用于以简单的阴影（平面或线框）方式绘制几何的材料。
        //  MeshLambertMaterial：无光泽表面材质，无镜面高光。
        //  MeshPhongMaterial：光泽表面材质，镜面高光，反射。
        let planeGeometry = new THREE.PlaneGeometry(1000, 1000, 10, 10);
        let plane = new THREE.Mesh(planeGeometry, new THREE.MeshLambertMaterial({
          color: 0x202020,
          // opacity: 0.5,
          // transparent: true,
          side: THREE.DoubleSide//两面可见
        }));
        plane.position.x = 0;
        plane.position.y = 0;
        plane.position.z = index * 500;
        plane.layers.set(index);
        this.scene.add(plane);
        // label
        let earthDiv = document.createElement('div');
        earthDiv.className = `label th${index}`;
        earthDiv.textContent = `第${index}层`;
        earthDiv.style.marginTop = '-1em';
        let earthLabel = new CSS2DObject(earthDiv);
        earthLabel.position.set(0, 0, 0);
        plane.add(earthLabel); // 将标签添加到网格模型中

      },
      animate() {
        requestAnimationFrame(this.animate);
        this.controls.update(); // only required if controls.enableDamping = true, or if controls.autoRotate = true
        this.render();
        self.stats.begin();
        self.stats.end();
      },
      render() {
        let self = this;
        /*交互开始*/
        self.raycaster.setFromCamera(self.mouse, self.camera);
        let intersects = self.raycaster.intersectObjects(self.scene.children);
        // 有目标被选中
        if (intersects.length > 0) {
          if (self.InterSelected != intersects[0].object) {
            if (self.InterSelected)
              self.InterSelected.material.opacity = self.InterSelected.currentOpacity;

            self.InterSelected = intersects[0].object;
            self.InterSelected.currentOpacity = self.InterSelected.material.opacity;
            self.InterSelected.material.opacity = 1;
          }
        } else { // 没有目标被选中
          if (self.InterSelected) // 变为原来的样式
            self.InterSelected.material.opacity = self.InterSelected.currentOpacity;
          self.InterSelected = null;
        }
        /*交互结束*/
        if (self.flag && self.forceSate[0] && self.forceSate[1] && self.forceSate[2]) {
          self.flag = false;
          // 分别求相交的数据集合 并绘制出线
          self.commonNode(self.gData1, self.gData2);
          self.commonNode(self.gData1, self.gData3);
          self.commonNode(self.gData2, self.gData3);
        }
        self.renderer.render(self.scene, self.camera);
        self.labelRenderer.render(self.scene, self.camera);
      },
      onWindowResize() {
        let self = this;
        self.camera.aspect = window.innerWidth / window.innerHeight;
        self.camera.updateProjectionMatrix();
        self.renderer.setSize(window.innerWidth, window.innerHeight);
      },
      guiInit() {
        let self = this;
        let layers = {第0层: true, 第1层: true, 第2层: true};
        const gui = new dat.GUI();
        gui.add(layers, '第0层').onChange(value => {
          self.camera.layers.toggle(0);
          self.lableToggle(0, value);
          self.lineToggle(0, value);
        });
        gui.add(layers, '第1层').onChange(value => {
          self.camera.layers.toggle(1);
          self.lableToggle(1, value);
          self.lineToggle(1, value);
        });
        gui.add(layers, '第2层').onChange(value => {
          self.camera.layers.toggle(2);
          self.lableToggle(2, value);
          self.lineToggle(2, value);
        });
      },
      statsInit() {
        self.stats = new Stats();
        self.stats.showPanel(0);  // 0: fps, 1: ms, 2: mb, 3+: custom
        document.body.appendChild(stats.dom);
      },
      onDocumentMouseMove(event) {
        let self = this;
        event.preventDefault();
        self.mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        self.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
      },
      /*切换标签*/
      lableToggle(layer, isShow) {
        isShow === true ?
          document.getElementsByClassName(`label th${layer}`)[0].style.visibility = 'visible' :
          document.getElementsByClassName(`label th${layer}`)[0].style.visibility = 'hidden'
      },
      computed() {
        let self = this;
        const N = 100;
        // 0- N不包括N
        this.gData1 = {
          // 生成从0-N 不包括N的数组
          nodes: [...Array(N).keys()].map(i => ({id: i, groups: 0})),
          // filter过滤id为0的数值
          // Math.random()  [0,1)
          // Math.round(x)
          links: [...Array(N).keys()]
            .filter(id => id)
            .map(id => ({
              source: id,
              target: Math.round(Math.random() * (id - 1))
            }))
        };

        this.gData2 = {
          nodes: [...Array(N).keys()].map(i => ({id: i + 90, groups: 1})),
          links: [...Array(N).keys()]
            .filter(id => id)
            .map(id => ({
              source: id + 90,
              target: Math.round(Math.random() * (id - 1)) + 90
            }))
        };

        this.gData3 = {
          nodes: [...Array(N).keys()].map(i => ({id: i + 180, groups: 2})),
          links: [...Array(N).keys()]
            .filter(id => id)
            .map(id => ({
              source: id + 180,
              target: Math.round(Math.random() * (id - 1)) + 180
            }))
        };

        // 分别使用力导向图
        self.forceLayout(self.gData1, 0);
        self.forceLayout(self.gData2, 1);
        self.forceLayout(self.gData3, 2);
      },
      forceLayout(data, index) {
        let self = this;
        const simulation = d3.forceSimulation()
          .force('link', d3.forceLink().id((d) => d.id))
          .force('charge', d3.forceManyBody())
          .force('center', d3.forceCenter(0, 0));

        let ticked = function () {
          if (simulation.alpha() <= 0.01) {
            self.addObj(data, index);
            simulation.stop(); // 停止力模拟
            self.forceSate[index] = true;
          }
        };
        simulation
          .nodes(data.nodes)
          .on('tick', ticked);

        simulation.force('link')
          .links(data.links);
      },
      getColor() {
        // 要成为一个全局函数 每个比例尺都是独立的
        return d3.scaleOrdinal(d3.schemeCategory10);
      },
      commonNode(data1, data2) {
        let self = this;
        let arr1 = data1.nodes.map(item => item.id);
        let arr2 = data2.nodes.map(item => item.id);
        let interaction = arr1.filter(v => arr2.includes(v));
        interaction.forEach(item => {
          let position1 = {}, position2 = {};
          for (let item1 of data1.nodes) {
            if (item1.id == item) {
              position1 = {x: item1.x, y: item1.y, z: item1.groups * 500, groups: item1.groups};
              break;
            }
          }
          for (let item2 of data2.nodes) {
            if (item2.id == item) {
              position2 = {x: item2.x, y: item2.y, z: item2.groups * 500, groups: item2.groups}
              break;
            }
          }
          self.drawLine(position1, position2);
        })
      },
      drawLine(p1, p2) {
        let self = this;
        let material = new THREE.LineBasicMaterial({
          opacity: 0.5,
          transparent: true,
          vertexColors: true // 支持颜色插值
        });
        let color1 = new THREE.Color(self.color(p1.groups)), color2 = new THREE.Color(self.color(p2.groups));

        // 声明一个几何体 给几何体添加位置节点和颜色
        let geometry = new THREE.Geometry();
        geometry.vertices.push(new THREE.Vector3(p1.x, p1.y, p1.z));
        geometry.vertices.push(new THREE.Vector3(p2.x, p2.y, p2.z));
        geometry.colors.push(color1, color2);
        let line = new THREE.Line(geometry, material);
        line.name = `${p1.groups}_${p2.groups}`;
        line.layers.set(3); //  要设置为3 默认为0
        this.scene.add(line);
      },
      /*切换线条*/
      lineToggle(index, value) {
        console.log(index);
        let self = this;
        self.scene.children.forEach(item => {
          if (item.name.includes(`${index}`)) {
            item.visible = value;
          }
        });
      },
    }
  }
</script>

<style scoped>
  #WebGL-output {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 0; /* to not conflict with dat.gui */
  }

  #WebGL-output >>> .label {
    color: #FFF;
    font-family: sans-serif;
    padding: 2px;
    background: rgba(0, 0, 0, .6);
  }
</style>
