#### 镜面高光和载入json模型
<p><iframe style="width: 100%; height: 520px; text-align:center;" src="/nehe_webgl/lessons/NeHeWebGL22.html" frameborder="0" width="500" height="500"></iframe></p>

#### 源码笔记

```
<html>

<head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>WebGL中文教程 - 镜面高光和载入json模型！</title>

<script type="text/javascript" src="../lib/Oak3D_v_0_5.js"></script>

<script id="per-fragment-lighting-fs" type="x-shader/x-fragment">

    precision mediump float;


    varying vec2 vTextureCoord;
    varying vec3 vTransformedNormal;
    varying vec4 vPosition;

    uniform float uMaterialShininess;

    uniform bool uShowSpecularHighlights;
    uniform bool uUseLighting;
    uniform bool uUseTextures;

    uniform vec3 uAmbientColor;

    uniform vec3 uPointLightingLocation;
    uniform vec3 uPointLightingSpecularColor;
    uniform vec3 uPointLightingDiffuseColor;

    uniform sampler2D uSampler;


    void main(void) {
        vec3 lightWeighting;
        //是否开启光照效果
        if (!uUseLighting) {
            lightWeighting = vec3(1.0, 1.0, 1.0);
        } else {
            //计算光照的方向
            vec3 lightDirection = normalize(uPointLightingLocation - vPosition.xyz);
            vec3 normal = normalize(vTransformedNormal);

            //用来存储镜面高光的带来的亮度
            float specularLightWeighting = 0.0;
            //使用光照的同时开启镜面高光效果
            if (uShowSpecularHighlights) {
                //首先得到观察方向上的单位向量V
                vec3 eyeDirection = normalize(-vPosition.xyz);
                //计算出来反射方向
                vec3 reflectionDirection = reflect(-lightDirection, normal);

                specularLightWeighting = pow(max(dot(reflectionDirection, eyeDirection), 0.0), uMaterialShininess);
            }

            float diffuseLightWeighting = max(dot(normal, lightDirection), 0.0);
            lightWeighting = uAmbientColor
                + uPointLightingSpecularColor * specularLightWeighting
                + uPointLightingDiffuseColor * diffuseLightWeighting;
        }

        vec4 fragmentColor;
        //是否使用纹理
        if (uUseTextures) {
            fragmentColor = texture2D(uSampler, vec2(vTextureCoord.s, vTextureCoord.t));
        } else {
            fragmentColor = vec4(1.0, 1.0, 1.0, 1.0);
        }
        gl_FragColor = vec4(fragmentColor.rgb * lightWeighting, fragmentColor.a);
    }
</script>

<script id="per-fragment-lighting-vs" type="x-shader/x-vertex">
    attribute vec3 aVertexPosition;
    attribute vec3 aVertexNormal;
    attribute vec2 aTextureCoord;

    uniform mat4 uMVMatrix;
    uniform mat4 uPMatrix;
    uniform mat4 uNMatrix;

    varying vec2 vTextureCoord;
    varying vec3 vTransformedNormal;
    varying vec4 vPosition;


    void main(void) {
        vPosition = uMVMatrix * vec4(aVertexPosition, 1.0);
        gl_Position = uPMatrix * vPosition;
        vTextureCoord = aTextureCoord;
        vTransformedNormal = (uNMatrix * vec4(aVertexNormal , 1.0)).xyz;
    }
</script>


<script type="text/javascript">

    var gl;

    function initGL(canvas) {
        try {
            gl = canvas.getContext("experimental-webgl");
            gl.viewportWidth = canvas.width;
            gl.viewportHeight = canvas.height;
        } catch (e) {
        }
        if (!gl) {
            alert("Could not initialise WebGL, sorry :-(");
        }
    }


    function getShader(gl, id) {
        var shaderScript = document.getElementById(id);
        if (!shaderScript) {
            return null;
        }

        var str = "";
        var k = shaderScript.firstChild;
        while (k) {
            if (k.nodeType == 3) {
                str += k.textContent;
            }
            k = k.nextSibling;
        }

        var shader;
        if (shaderScript.type == "x-shader/x-fragment") {
            shader = gl.createShader(gl.FRAGMENT_SHADER);
        } else if (shaderScript.type == "x-shader/x-vertex") {
            shader = gl.createShader(gl.VERTEX_SHADER);
        } else {
            return null;
        }

        gl.shaderSource(shader, str);
        gl.compileShader(shader);

        if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
            alert(gl.getShaderInfoLog(shader));
            return null;
        }

        return shader;
    }


    var shaderProgram;

    function initShaders() {
        var fragmentShader = getShader(gl, "per-fragment-lighting-fs");
        var vertexShader = getShader(gl, "per-fragment-lighting-vs");

        shaderProgram = gl.createProgram();
        gl.attachShader(shaderProgram, vertexShader);
        gl.attachShader(shaderProgram, fragmentShader);
        gl.linkProgram(shaderProgram);

        if (!gl.getProgramParameter(shaderProgram, gl.LINK_STATUS)) {
            alert("Could not initialise shaders");
        }

        gl.useProgram(shaderProgram);

        shaderProgram.vertexPositionAttribute = gl.getAttribLocation(shaderProgram, "aVertexPosition");
        gl.enableVertexAttribArray(shaderProgram.vertexPositionAttribute);

        shaderProgram.vertexNormalAttribute = gl.getAttribLocation(shaderProgram, "aVertexNormal");
        gl.enableVertexAttribArray(shaderProgram.vertexNormalAttribute);

        shaderProgram.textureCoordAttribute = gl.getAttribLocation(shaderProgram, "aTextureCoord");
        gl.enableVertexAttribArray(shaderProgram.textureCoordAttribute);

        shaderProgram.pMatrixUniform = gl.getUniformLocation(shaderProgram, "uPMatrix");
        shaderProgram.mvMatrixUniform = gl.getUniformLocation(shaderProgram, "uMVMatrix");
        shaderProgram.nMatrixUniform = gl.getUniformLocation(shaderProgram, "uNMatrix");
        shaderProgram.samplerUniform = gl.getUniformLocation(shaderProgram, "uSampler");
        shaderProgram.materialShininessUniform = gl.getUniformLocation(shaderProgram, "uMaterialShininess");
        shaderProgram.showSpecularHighlightsUniform = gl.getUniformLocation(shaderProgram, "uShowSpecularHighlights");
        shaderProgram.useTexturesUniform = gl.getUniformLocation(shaderProgram, "uUseTextures");
        shaderProgram.useLightingUniform = gl.getUniformLocation(shaderProgram, "uUseLighting");
        shaderProgram.ambientColorUniform = gl.getUniformLocation(shaderProgram, "uAmbientColor");
        shaderProgram.pointLightingLocationUniform = gl.getUniformLocation(shaderProgram, "uPointLightingLocation");
        shaderProgram.pointLightingSpecularColorUniform = gl.getUniformLocation(shaderProgram, "uPointLightingSpecularColor");
        shaderProgram.pointLightingDiffuseColorUniform = gl.getUniformLocation(shaderProgram, "uPointLightingDiffuseColor");
    }


    function handleLoadedTexture(texture) {
        gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
        gl.bindTexture(gl.TEXTURE_2D, texture);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, texture.image);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_NEAREST);
        gl.generateMipmap(gl.TEXTURE_2D);

        gl.bindTexture(gl.TEXTURE_2D, null);
    }


    var earthTexture;
    var galvanizedTexture;

    function initTextures() {
        earthTexture = gl.createTexture();
        earthTexture.image = new Image();
        earthTexture.image.onload = function () {
            handleLoadedTexture(earthTexture)
        }
        earthTexture.image.src = "../resources/earth/earth.jpg";

        galvanizedTexture = gl.createTexture();
        galvanizedTexture.image = new Image();
        galvanizedTexture.image.onload = function () {
            handleLoadedTexture(galvanizedTexture)
        }
        galvanizedTexture.image.src = "../resources/earth/texture.bmp";
    }


    var mvMatrix ;
    var mvMatrixStack = [];
    var pMatrix ;

    function mvPushMatrix() {
        var copy = new okMat4();
        mvMatrix.clone(copy);
        mvMatrixStack.push(copy);
    }

    function mvPopMatrix() {
        if (mvMatrixStack.length == 0) {
            throw "Invalid popMatrix!";
        }
        mvMatrix = mvMatrixStack.pop();
    }


    function setMatrixUniforms() {
        gl.uniformMatrix4fv(shaderProgram.pMatrixUniform, false, pMatrix.toArray());
        gl.uniformMatrix4fv(shaderProgram.mvMatrixUniform, false, mvMatrix.toArray());
        
        var normalMatrix = mvMatrix.inverse().transpose();
        gl.uniformMatrix4fv(shaderProgram.nMatrixUniform, false, normalMatrix.toArray());
    }

	
	
	
	
	
	
    function degToRad(degrees) {
        return degrees * Math.PI / 180;
    }




    //在这里把json数据格式传过来，我来开始解析数据了
    var teapotVertexPositionBuffer;
    var teapotVertexNormalBuffer;
    var teapotVertexTextureCoordBuffer;
    var teapotVertexIndexBuffer;



    //处理json数据格式
    //这个json文件中存储了顶点的位置信息，法线信息，纹理坐标和顶点索引信息
    function handleLoadedTeapot(teapotData) {

        //在这里把json中的数据循环打印出来
        /*for (var item in teapotData){
            alert(item.vertexNormals+"   \n" +
                "\n"+teapotData.vertexTextureCoords+"   \n" +
                ""+teapotData.vertexPositions);
        }*/
        //alert(teapotData.vertexNormals);



        //创建一个顶点法向量缓冲区对象
        teapotVertexNormalBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexNormalBuffer);
        //新建一个数组，直接把这一堆的法向量数据作为一个数组传进去， 绑定缓冲区到目标对象
        gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(teapotData.vertexNormals), gl.STATIC_DRAW);
        teapotVertexNormalBuffer.itemSize = 3;
        teapotVertexNormalBuffer.numItems = teapotData.vertexNormals.length / 3;


        teapotVertexTextureCoordBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexTextureCoordBuffer);
        gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(teapotData.vertexTextureCoords), gl.STATIC_DRAW);
        teapotVertexTextureCoordBuffer.itemSize = 2;
        teapotVertexTextureCoordBuffer.numItems = teapotData.vertexTextureCoords.length / 2;

        teapotVertexPositionBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexPositionBuffer);
        gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(teapotData.vertexPositions), gl.STATIC_DRAW);
        teapotVertexPositionBuffer.itemSize = 3;
        teapotVertexPositionBuffer.numItems = teapotData.vertexPositions.length / 3;

        teapotVertexIndexBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, teapotVertexIndexBuffer);
        gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array(teapotData.indices), gl.STATIC_DRAW);
        teapotVertexIndexBuffer.itemSize = 1;
        teapotVertexIndexBuffer.numItems = teapotData.indices.length;



        //载入数据完毕之后
        document.getElementById("loadingtext").textContent = "";
    }



    //这里使用json这种数据格式把文件载入进来
    function loadTeapot() {
        var request = new XMLHttpRequest();
        request.open("GET", "../resources/earth/Teapot.json");
        request.onreadystatechange = function () {
            if (request.readyState == 4) {
                //载入完毕之后，开始去解析这里的文本内容(这里把传递过来的文本信息转换为了一个个json对象)
                handleLoadedTeapot(JSON.parse(request.responseText));
            }
        }
        request.send();
    }



    //设置初始的角度
    var teapotAngle = 180;


    function drawScene() {
        gl.viewport(0, 0, gl.viewportWidth, gl.viewportHeight);
        gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

        if (teapotVertexPositionBuffer == null || teapotVertexNormalBuffer == null || teapotVertexTextureCoordBuffer == null || teapotVertexIndexBuffer == null) {
            return;
        }
        
        pMatrix = okMat4Proj(45, gl.viewportWidth / gl.viewportHeight, 0.1, 100.0);

        var specularHighlights = document.getElementById("specular").checked;
        gl.uniform1i(shaderProgram.showSpecularHighlightsUniform, specularHighlights);

        var lighting = document.getElementById("lighting").checked;
        gl.uniform1i(shaderProgram.useLightingUniform, lighting);
        if (lighting) {
            gl.uniform3f(
                shaderProgram.ambientColorUniform,
                parseFloat(document.getElementById("ambientR").value),
                parseFloat(document.getElementById("ambientG").value),
                parseFloat(document.getElementById("ambientB").value)
            );

            gl.uniform3f(
                shaderProgram.pointLightingLocationUniform,
                parseFloat(document.getElementById("lightPositionX").value),
                parseFloat(document.getElementById("lightPositionY").value),
                parseFloat(document.getElementById("lightPositionZ").value)
            );

            gl.uniform3f(
                shaderProgram.pointLightingSpecularColorUniform,
                parseFloat(document.getElementById("specularR").value),
                parseFloat(document.getElementById("specularG").value),
                parseFloat(document.getElementById("specularB").value)
            );

            gl.uniform3f(
                shaderProgram.pointLightingDiffuseColorUniform,
                parseFloat(document.getElementById("diffuseR").value),
                parseFloat(document.getElementById("diffuseG").value),
                parseFloat(document.getElementById("diffuseB").value)
            );
        }

        var texture = document.getElementById("texture").value;
        gl.uniform1i(shaderProgram.useTexturesUniform, texture != "none");

        mvMatrix = okMat4Trans(0, 0, -40);
        mvMatrix.rot(OAK.SPACE_LOCAL, 23.4, 1, 0, -1, true);
        mvMatrix.rotY(OAK.SPACE_LOCAL, teapotAngle, true);

        gl.activeTexture(gl.TEXTURE0);
        if (texture == "earth") {
            gl.bindTexture(gl.TEXTURE_2D, earthTexture);
        } else if (texture == "galvanized") {
            gl.bindTexture(gl.TEXTURE_2D, galvanizedTexture);
        }
        gl.uniform1i(shaderProgram.samplerUniform, 0);

        gl.uniform1f(shaderProgram.materialShininessUniform, parseFloat(document.getElementById("shininess").value));

        gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexPositionBuffer);
        gl.vertexAttribPointer(shaderProgram.vertexPositionAttribute, teapotVertexPositionBuffer.itemSize, gl.FLOAT, false, 0, 0);

        gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexTextureCoordBuffer);
        gl.vertexAttribPointer(shaderProgram.textureCoordAttribute, teapotVertexTextureCoordBuffer.itemSize, gl.FLOAT, false, 0, 0);

        gl.bindBuffer(gl.ARRAY_BUFFER, teapotVertexNormalBuffer);
        gl.vertexAttribPointer(shaderProgram.vertexNormalAttribute, teapotVertexNormalBuffer.itemSize, gl.FLOAT, false, 0, 0);

        gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, teapotVertexIndexBuffer);
        setMatrixUniforms();
        gl.drawElements(gl.TRIANGLES, teapotVertexIndexBuffer.numItems, gl.UNSIGNED_SHORT, 0);
    }


    var lastTime = 0;

    function animate() {
        var timeNow = new Date().getTime();
        if (lastTime != 0) {
            var elapsed = timeNow - lastTime;

            teapotAngle += 0.05 * elapsed;
        }
        lastTime = timeNow;
    }


    function tick() {
        okRequestAnimationFrame(tick);
        drawScene();
        animate();
    }


    function webGLStart() {
        var canvas = document.getElementById("lesson14-canvas");
        initGL(canvas);
        initShaders();
        initTextures();
        loadTeapot();

        gl.clearColor(0.0, 0.0, 0.0, 1.0);
        gl.enable(gl.DEPTH_TEST);

        tick();
    }

</script>


<style type="text/css">
    #loadingtext {
        position:absolute;
        top:250px;
        left:150px;
        font-size:2em;
        color: white;
    }
</style>


</head>


<body onload="webGLStart();">
<br>
    <canvas id="lesson14-canvas" style="border: none;" width="500" height="500"></canvas>

    <div id="loadingtext">Loading world...</div>
    <br/>

    <input type="checkbox" id="specular" checked />启用镜面高光<br/>
    <input type="checkbox" id="lighting" checked />启用光照<br/>

    纹理：
    <select id="texture">
        <option value="none">不使用纹理</option>
        <option selected value="galvanized">电镀金属</option>
        <option value="earth">地球</option>
    </select>
    <br/>


    <h2>材质：</h2>

    <table style="border: 0; padding: 10px;">
        <tr>
            <td><b>光泽度：</b>
            <td><input type="text" id="shininess" value="32.0" />
        </tr>
    </table>


    <h2>点光源：</h2>

    <table style="border: 0; padding: 10px;">
        <tr>
            <td><b>位置：</b>
            <td>X: <input type="text" id="lightPositionX" value="-10.0" />
            <td>Y: <input type="text" id="lightPositionY" value="4.0" />
            <td>Z: <input type="text" id="lightPositionZ" value="-20.0" />
        </tr>
        <tr>
            <td><b>镜面反射颜色：</b>
            <td>R: <input type="text" id="specularR" value="0.8" />
            <td>G: <input type="text" id="specularG" value="0.8" />
            <td>B: <input type="text" id="specularB" value="0.8" />
        </tr>
        <tr>
            <td><b>漫反射颜色：</b>
            <td>R: <input type="text" id="diffuseR" value="0.8" />
            <td>G: <input type="text" id="diffuseG" value="0.8" />
            <td>B: <input type="text" id="diffuseB" value="0.8" />
        </tr>
    </table>


    <h2>环境光：</h2>

    <table style="border: 0; padding: 10px;">
        <tr>
            <td><b>颜色：</b>
            <td>R: <input type="text" id="ambientR" value="0.2" />
            <td>G: <input type="text" id="ambientG" value="0.2" />
            <td>B: <input type="text" id="ambientB" value="0.2" />
        </tr>
    </table>
    <br/>

</body>

</html>

```