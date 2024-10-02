<script>
    import { onMount } from 'svelte';
    import * as THREE from 'three';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

    let container;
    let activeSphere, activeTexture;
    let activePanoramaIndex = 0;

    // Store the textures of multiple panorama images
    let panoramaTextures = [
        '/MainGateFlipped.jpg',   // First panorama
        '/Reception.jpg',   // Second panorama
        '/Corridor.jpg'      // Third panorama
    ];

    // Hotspot definitions for different panoramas
    let hotspotsByPanorama = {
        0: [getHotspot(200, 0.0, Math.PI / 2, 1)],  // First panorama hotspot (move right)
        1: [
            getHotspot(200, 0.0, Math.PI / 2, 2),   // Second panorama to third
            getHotspot(200, 3.0, Math.PI / 2, 0)    // Second panorama back to first
        ],
        2: [getHotspot(200, 3.0, Math.PI / 2, 1)]   // Third panorama back to second
    };

    // Helper function to calculate spherical coordinates and return hotspot
    function getHotspot(radius, theta, phi, targetPanoramaIndex) {
        let x = radius * Math.sin(phi) * Math.cos(theta);
        let y = radius * Math.cos(phi);
        let z = radius * Math.sin(phi) * Math.sin(theta);
        return { position: { x: x, y: y, z: z }, targetPanoramaIndex };
    }

    // Load Texture function
    const loadTexture = (url) => {
        return new Promise((resolve) => {
            const textureLoader = new THREE.TextureLoader();
            textureLoader.load(url, resolve);
        });
    };

    onMount(async () => {
        // Set up the scene
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 0, 0.1);

        const renderer = new THREE.WebGLRenderer({ alpha: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        container.appendChild(renderer.domElement);

        // Load initial texture (the first panorama)
        activeTexture = await loadTexture(panoramaTextures[0]);
        createSphere(scene, activeTexture);

        // Set up Orbit Controls
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.25;
        controls.rotateSpeed = -0.5;

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

            // Calculate mouse position in normalized device coordinates (-1 to +1)
            mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
            mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

            // Raycast from the camera to find intersections with the hotspots
            raycaster.setFromCamera(mouse, camera);
            let intersects = raycaster.intersectObjects(scene.children);

            if (intersects.length > 0) {
                let hotspot = intersects.find(intersect => intersect.object.isHotspot);
                if (hotspot) {
                    transitionToPanorama(hotspot.object.targetPanoramaIndex);
                }
            }
        }

        // Load new panorama and transition to it
        async function transitionToPanorama(panoramaIndex) {
            let newTexture = await loadTexture(panoramaTextures[panoramaIndex]);

            // Smooth transition (fade out and in)
            activeSphere.material.opacity = 0; // Fade out old panorama
            setTimeout(() => {
                activeSphere.material.map = newTexture;
                activeSphere.material.opacity = 1; // Fade in new panorama
                activeTexture = newTexture;
                activePanoramaIndex = panoramaIndex; // Update the active panorama index
                updateHotspots(scene);  // Update hotspots for the new panorama
            }, 500); // Time for fade transition
        }

        // Helper function to create the 360 sphere
        function createSphere(scene, texture) {
            const geometry = new THREE.SphereGeometry(500, 60, 40);
            const material = new THREE.MeshBasicMaterial({
                map: texture,
                side: THREE.BackSide,
                transparent: true,
                opacity: 1  // Set opacity to allow fade transitions
            });

            activeSphere = new THREE.Mesh(geometry, material);
            scene.add(activeSphere);
        }

        // Create hotspots on the sphere for the current panorama
        function createHotspots(scene) {
            // Retrieve relevant hotspots for the active panorama
            let hotspots = hotspotsByPanorama[activePanoramaIndex];
            hotspots.forEach((hotspot) => {
                const hotspotGeometry = new THREE.SphereGeometry(5, 32, 32);
                const hotspotMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
                const hotspotMesh = new THREE.Mesh(hotspotGeometry, hotspotMaterial);
                hotspotMesh.position.set(hotspot.position.x, hotspot.position.y, hotspot.position.z);

                hotspotMesh.isHotspot = true; // Custom flag to identify hotspot
                hotspotMesh.targetPanoramaIndex = hotspot.targetPanoramaIndex; // Which panorama this leads to

                scene.add(hotspotMesh);
            });
        }

        // Remove previous hotspots and create new ones when transitioning between panoramas
        function updateHotspots(scene) {
            // Remove old hotspots
            scene.children = scene.children.filter(child => !child.isHotspot);
            // Create new hotspots
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
