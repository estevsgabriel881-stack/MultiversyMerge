
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Multiversy</title>
    

    <style>
        body { margin: 0; background: #000; overflow: hidden; }
        
        /* BARRA DE LEGENDA NO TOPO */
        #barra-legenda {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 50px;
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(8px);
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 40px;
            z-index: 100;
            border-bottom: 1px solid rgba(255, 255, 255, 0.2);
            pointer-events: none; /* Deixa cliques passarem para o universo */
        }

        .item-legenda {
            display: flex;
            align-items: center;
            gap: 10px;
            color: white;
            font-family: 'Segoe UI', sans-serif;
            font-size: 14px;
            letter-spacing: 1px;
        }

        .quadrado { width: 16px; height: 16px; border-radius: 3px; border: 1px solid #fff; }
        .cinza { background: #888; }
        .azul { background: #0077be; }

        #guia {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: white; font-family: 'Segoe UI', sans-serif; letter-spacing: 5px;
            text-transform: uppercase; opacity: 0.3; pointer-events: none;
            z-index: 10;
        }
    </style>
</head>
<body>

    

    <div id="guia">Mova o mouse ou clique</div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

    <script>
        let cena, camera, renderizador, estrelas, planetas = [], nebulosas = [], buracosNegros = [], galaxias = [];
        let starPositions;

        const blackHoleStarShader = {
            uniforms: {
                'pointSize': { value: 2.0 },
                'blackHolePos': { value: new THREE.Vector3(0, 0, 0) },
                'blackHoleRadius': { value: 0.0 }
            },
            vertexShader: `
                uniform float pointSize;
                uniform vec3 blackHolePos;
                uniform float blackHoleRadius;
                void main() {
                    vec3 vPosition = position;
                    float distToBlackHole = distance(vPosition, blackHolePos);
                    if (distToBlackHole < blackHoleRadius && blackHoleRadius > 0.1) {
                        vec3 dirToBlackHole = normalize(blackHolePos - vPosition);
                        float pullFactor = 1.0 - pow(distToBlackHole / blackHoleRadius, 0.5); 
                        vPosition += dirToBlackHole * pullFactor * blackHoleRadius * 0.1;
                    }
                    vec4 mvPosition = modelViewMatrix * vec4(vPosition, 1.0);
                    gl_PointSize = pointSize * (1000.0 / -mvPosition.z);
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
            fragmentShader: `void main() { gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0); }`
        };

        function iniciar() {
            cena = new THREE.Scene();
            camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 1, 10000);
            camera.position.z = 1000;

            renderizador = new THREE.WebGLRenderer({ antialias: true });
            renderizador.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderizador.domElement);

            const luzPrincipal = new THREE.DirectionalLight(0xffffff, 1.2);
            luzPrincipal.position.set(5, 3, 5);
            cena.add(luzPrincipal);
            cena.add(new THREE.AmbientLight(0x333333));

            const geometriaEstrelas = new THREE.BufferGeometry();
            const pos = [];
            for (let i = 0; i < 15000; i++) {
                pos.push(THREE.MathUtils.randFloatSpread(4000), THREE.MathUtils.randFloatSpread(4000), THREE.MathUtils.randFloatSpread(4000));
            }
            starPositions = new THREE.Float32BufferAttribute(pos, 3);
            geometriaEstrelas.setAttribute('position', starPositions);
            
            const materialEstrelas = new THREE.ShaderMaterial({
                uniforms: blackHoleStarShader.uniforms,
                vertexShader: blackHoleStarShader.vertexShader,
                fragmentShader: blackHoleStarShader.fragmentShader,
                transparent: true,
                blending: THREE.AdditiveBlending
            });
            
            estrelas = new THREE.Points(geometriaEstrelas, materialEstrelas);
            cena.add(estrelas);

            animar();

            setInterval(criarBuracoNegro, 5 * 60 * 1000);
            setInterval(criarGalaxia, 2 * 60 * 1000);
        }

        function gerarTexturaPlaneta() {
            const canvas = document.createElement('canvas');
            canvas.width = 512; canvas.height = 256;
            const ctx = canvas.getContext('2d');
            ctx.fillStyle = `hsl(${Math.random() * 360}, 20%, 15%)`;
            ctx.fillRect(0, 0, 512, 256);
            for (let i = 0; i < 60; i++) {
                const x = Math.random() * 512, y = Math.random() * 256, raio = Math.random() * 80 + 20;
                const grad = ctx.createRadialGradient(x, y, 0, x, y, raio);
                grad.addColorStop(0, `hsla(${Math.random() * 360}, 50%, 40%, 0.4)`);
                grad.addColorStop(1, 'transparent');
                ctx.fillStyle = grad; ctx.beginPath(); ctx.arc(x, y, raio, 0, Math.PI * 2); ctx.fill();
            }
            return new THREE.CanvasTexture(canvas);
        }

        function gerarTexturaNebulosa() {
            const canvas = document.createElement('canvas');
            canvas.width = 128; canvas.height = 128;
            const contexto = canvas.getContext('2d');
            const gradiente = contexto.createRadialGradient(64, 64, 0, 64, 64, 64);
            gradiente.addColorStop(0, 'rgba(255, 255, 255, 1)');
            gradiente.addColorStop(0.2, 'rgba(255, 255, 255, 0.5)');
            gradiente.addColorStop(1, 'rgba(255, 255, 255, 0)');
            contexto.fillStyle = gradiente;
            contexto.fillRect(0, 0, 128, 128);
            return new THREE.CanvasTexture(canvas);
        }
        const texturaNebulosaBase = gerarTexturaNebulosa();

        function criarNebulosa(x, y, corPersonalizada = null, opacidade = 0.2, escala = 1.0) {
            const material = new THREE.SpriteMaterial({
                map: texturaNebulosaBase,
                color: corPersonalizada || new THREE.Color(`hsl(${Math.random() * 360}, 50%, 50%)`),
                transparent: true,
                opacity: opacidade,
                blending: THREE.AdditiveBlending
            });
            const nebulosa = new THREE.Sprite(material);
            if (x === undefined) { x = THREE.MathUtils.randFloatSpread(2000); y = THREE.MathUtils.randFloatSpread(2000); }
            const vetor = new THREE.Vector3((x / window.innerWidth) * 2 - 1, -(y / window.innerHeight) * 2 + 1, 0.5);
            vetor.unproject(camera);
            const dir = vetor.sub(camera.position).normalize();
            const pos = camera.position.clone().add(dir.multiplyScalar(-camera.position.z / dir.z));
            nebulosa.position.set(pos.x, pos.y, pos.z - 600);
            nebulosa.scale.set((500 + Math.random() * 500) * escala, (500 + Math.random() * 500) * escala, 1);
            cena.add(nebulosa);
            nebulosas.push({ mesh: nebulosa, velocidade: 1.2 });
            return nebulosa;
        }

        function criarPlaneta(x, y) {
            const vetor = new THREE.Vector3((x / window.innerWidth) * 2 - 1, -(y / window.innerHeight) * 2 + 1, 0.5);
            vetor.unproject(camera);
            const dir = vetor.sub(camera.position).normalize();
            const pos = camera.position.clone().add(dir.multiplyScalar(-camera.position.z / dir.z));
            const geo = new THREE.SphereGeometry(Math.random() * 45 + 15, 64, 64);
            const mat = new THREE.MeshPhongMaterial({ map: gerarTexturaPlaneta(), shininess: 5 });
            const planeta = new THREE.Mesh(geo, mat);
            planeta.position.set(pos.x, pos.y, pos.z - 500);
            cena.add(planeta);
            planetas.push({ mesh: planeta, velocidade: Math.random() * 4 + 2, rotacao: (Math.random() - 0.5) * 0.02 });
        }

        function criarBuracoNegro() {
            const grupoBH = new THREE.Group();
            const core = new THREE.Mesh(new THREE.SphereGeometry(60, 32, 32), new THREE.MeshBasicMaterial({ color: 0x000000 }));
            grupoBH.add(core);
            const ring = new THREE.Mesh(new THREE.TorusGeometry(100, 15, 2, 100), new THREE.MeshBasicMaterial({ color: 0xffaa00, transparent: true, opacity: 0.8, blending: THREE.AdditiveBlending }));
            ring.rotation.x = Math.PI / 2;
            grupoBH.add(ring);
            grupoBH.position.set(THREE.MathUtils.randFloatSpread(1000), THREE.MathUtils.randFloatSpread(1000), -5000);
            cena.add(grupoBH);
            buracosNegros.push({ mesh: grupoBH, velocidade: 1.5, raioInfluencia: 600 });
        }

        function criarGalaxia() {
            const grupoGalaxia = new THREE.Group();
            const centroZ = -6000 - Math.random() * 2000;
            for (let i = 0; i < 500; i++) {
                const star = new THREE.Mesh(new THREE.SphereGeometry(0.5, 4, 4), new THREE.MeshBasicMaterial({ color: 0xffffff }));
                const angle = i * 0.1; const r = i * 1.5;
                star.position.set(Math.cos(angle) * r, Math.sin(angle) * r, THREE.MathUtils.randFloatSpread(50));
                grupoGalaxia.add(star);
            }
            grupoGalaxia.position.set(THREE.MathUtils.randFloatSpread(2000), THREE.MathUtils.randFloatSpread(2000), centroZ);
            cena.add(grupoGalaxia);
            galaxias.push({ mesh: grupoGalaxia, velocidade: 0.5, rotacao: 0.001 });
        }

        window.addEventListener('mousedown', (e) => {
            // Agora qualquer clique cria planetas, já que não há botões para ignorar
            criarPlaneta(e.clientX, e.clientY);
            if(Math.random() > 0.3) criarNebulosa(e.clientX, e.clientY);
        });

        window.addEventListener('mousemove', (e) => {
            camera.position.x += (e.clientX - window.innerWidth / 2) * 0.005;
            camera.position.y += -(e.clientY - window.innerHeight / 2) * 0.005;
            camera.lookAt(cena.position);
        });

        function animar() {
            requestAnimationFrame(animar);
            if (buracosNegros.length > 0) {
                blackHoleStarShader.uniforms.blackHolePos.value.copy(buracosNegros[0].mesh.position);
                blackHoleStarShader.uniforms.blackHoleRadius.value = buracosNegros[0].raioInfluencia;
            } else { blackHoleStarShader.uniforms.blackHoleRadius.value = 0; }

            estrelas.rotation.y += 0.0008;
            planetas.forEach((p, i) => { 
                p.mesh.position.z += p.velocidade; p.mesh.rotation.y += p.rotacao;
                buracosNegros.forEach(bh => {
                    if(p.mesh.position.distanceTo(bh.mesh.position) < bh.raioInfluencia) {
                        p.mesh.position.lerp(bh.mesh.position, 0.02); p.mesh.scale.multiplyScalar(0.99);
                    }
                });
                if(p.mesh.position.z > 1100 || p.mesh.scale.x < 0.1) { cena.remove(p.mesh); planetas.splice(i,1); }
            });

            buracosNegros.forEach((bh, i) => { 
                bh.mesh.position.z += bh.velocidade; bh.mesh.children[1].rotation.z += 0.05;
                if(bh.mesh.position.z > 1100) { cena.remove(bh.mesh); buracosNegros.splice(i,1); }
            });

            galaxias.forEach((g, i) => { 
                g.mesh.position.z += g.velocidade; g.mesh.rotation.z += g.rotacao;
                if(g.mesh.position.z > 1100) { cena.remove(g.mesh); galaxias.splice(i,1); }
            });

            nebulosas.forEach((n, i) => {
                n.mesh.position.z += n.velocidade;
                if(n.mesh.position.z > 1100) { cena.remove(n.mesh); nebulosas.splice(i,1); }
            });

            renderizador.render(cena, camera);
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderizador.setSize(window.innerWidth, window.innerHeight);
        });

        iniciar();
    </script>

    <button class="btn-voltar" onclick="location.href='index.html'">⬅ Voltar ao Perfil</button>
    <p>Você saiu da plataforma principal para entrar no Big Bang.</p>




</body>
</head>
</html>
