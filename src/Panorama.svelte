<script>
    import { onMount } from 'svelte';
    import * as THREE from 'three';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

    let container;
    let activeSphere;
    let activePanoramaIndex = 0;
    let textures = [];  // Array to hold preloaded textures

    let panoramaTextures = [
        '/Ground Floor/MainGate.jpg', //0
        '/Ground Floor/Reception.jpg', //1
        '/Ground Floor/Corridor.jpg', //2
        '/Ground Floor/Stairs1.jpg', //3
        '/Ground Floor/Stairs2.jpg', //4
        '/Ground Floor/Assembly1.jpg', //5
        '/Ground Floor/Assembly2.jpg', //6
        '/Ground Floor/BadmintonCourt.jpg', //7
        '/Ground Floor/BankGate.jpg', //8
        '/Ground Floor/Canteen.jpg', //9
        '/Ground Floor/CarromRoom.jpg', //10
        '/Ground Floor/Field.jpg', //11
        '/Ground Floor/Stairs3.jpg', //12
        '/Ground Floor/TTRoom.jpg' //13
    ];

    let hotspotsByPanorama = {
        0: [getHotspot(200, 0.0, Math.PI / 2, 1)],
        1: [
            getHotspot(200, 0.0, Math.PI / 2, 2),
            getHotspot(200, 3.0, Math.PI / 2, 0)
        ],
        2: [
            getHotspot(200, 3.0, Math.PI / 2, 1),
            getHotspot(200, 0.0, Math.PI / 2, 7),
            getHotspot(200, 1.5, Math.PI / 2, 4),
            getHotspot(200, 4.6, Math.PI / 2, 3),
        ],
        3:[
            getHotspot(200, 3.0, Math.PI / 2, 2),
            getHotspot(200, 1.5, Math.PI / 2, 5)
        ],
        4:[
            getHotspot(200, 4.5, Math.PI / 2, 2),
            getHotspot(200, 0.0, Math.PI / 2, 8)
        ]
    };

    // Helper to calculate spherical coordinates
    function getHotspot(radius, theta, phi, targetPanoramaIndex) {
        let x = radius * Math.sin(phi) * Math.cos(theta);
        let y = radius * Math.cos(phi);
        let z = radius * Math.sin(phi) * Math.sin(theta);
        return { position: { x: x, y: y, z: z }, targetPanoramaIndex };
    }

    // Preload all panorama textures
    const preloadTextures = async () => {
        const textureLoader = new THREE.TextureLoader();
        const promises = panoramaTextures.map(url => loadTexture(url));
        textures = await Promise.all(promises);
    };

    const loadTexture = (url) => {
        return new Promise((resolve) => {
            const textureLoader = new THREE.TextureLoader();
            textureLoader.load(url, resolve);
        });
    };

    onMount(async () => {
    await preloadTextures(); // Preload all textures

    // Set up the scene
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    
    // Set the starting position of the camera (near the center)
    camera.position.set(0, 0, 0.1);
    
    // Optionally adjust the initial view direction (rotation in radians)
    camera.rotation.y = Math.PI / 2; // Rotate 90 degrees on the Y-axis (horizontal direction)
    camera.rotation.x = 0; // Set vertical angle (up or down)

    const renderer = new THREE.WebGLRenderer({ alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    container.appendChild(renderer.domElement);

    // Use the preloaded texture for the initial panorama
    createSphere(scene, textures[0]);

    // Set up Orbit Controls
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.25;
    controls.rotateSpeed = -0.5;

    // You can adjust the initial target of the camera
    controls.target.set(4, 0.5, 0);  // Pointing the camera towards a specific direction

    // Create initial hotspots (for the first panorama)
    createHotspots(scene);

    // Animation loop
    const animate = function () {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
    };

    animate();

    // Handle window resize
    window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    });

    // Handle hotspot clicks (in real 3D coordinates)
    document.addEventListener('mousedown', onDocumentMouseDown, false);
    let raycaster = new THREE.Raycaster();
    let mouse = new THREE.Vector2();

    function onDocumentMouseDown(event) {
        event.preventDefault();

        mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        raycaster.setFromCamera(mouse, camera);
        let intersects = raycaster.intersectObjects(scene.children);

        if (intersects.length > 0) {
            let hotspot = intersects.find(intersect => intersect.object.isHotspot);
            if (hotspot) {
                transitionToPanorama(hotspot.object.targetPanoramaIndex);
            }
        }
    }

    // Transition to preloaded panorama
    function transitionToPanorama(panoramaIndex) {
        activeSphere.material.opacity = 0;  // Fade out old panorama
        setTimeout(() => {
            activeSphere.material.map = textures[panoramaIndex];  // Use preloaded texture
            activeSphere.material.opacity = 1;  // Fade in new panorama
            activePanoramaIndex = panoramaIndex;
            updateHotspots(scene);  // Update hotspots
        }, 500);  // Smooth transition
    }

    function createSphere(scene, texture) {
        const geometry = new THREE.SphereGeometry(500, 60, 40);
        const material = new THREE.MeshBasicMaterial({
            map: texture,
            side: THREE.BackSide,
            transparent: true,
            opacity: 1
        });

        activeSphere = new THREE.Mesh(geometry, material);
        scene.add(activeSphere);
    }

    function createHotspots(scene) {
        let hotspots = hotspotsByPanorama[activePanoramaIndex];
        hotspots.forEach((hotspot) => {
            const hotspotGeometry = new THREE.SphereGeometry(5, 32, 32);
            const hotspotMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
            const hotspotMesh = new THREE.Mesh(hotspotGeometry, hotspotMaterial);
            hotspotMesh.position.set(hotspot.position.x, hotspot.position.y, hotspot.position.z);

            hotspotMesh.isHotspot = true;
            hotspotMesh.targetPanoramaIndex = hotspot.targetPanoramaIndex;

            scene.add(hotspotMesh);
        });
    }

    function updateHotspots(scene) {
        scene.children = scene.children.filter(child => !child.isHotspot);
        createHotspots(scene);
    }
});

</script>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        overflow: hidden; /* Prevent scrollbars */
        height: 100vh;    /* Ensure body takes full viewport height */
    }

    #container {
        display: flex;             /* Use flexbox for centering */
        justify-content: center;   /* Center horizontally */
        align-items: center;       /* Center vertically */
        width: 100%;               /* Full width */
        height: 100%;              /* Full height */
        background-color: black;   /* Optional: background color */
    }

    :global(canvas) {
        display: block;            /* Prevent extra space below the canvas */
        width: 100%;               /* Full width for canvas */
        height: 100%;              /* Full height for canvas */
    }
</style>

<div id="container" bind:this={container}></div>