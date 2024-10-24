<script>
    import { onMount } from 'svelte';
    import * as THREE from 'three';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

    let container;
    let activeSphere;
    let activePanoramaIndex = 0;
    let textures = [];  // Array to hold preloaded textures
    let hotspotMeshes = []; // Array to keep track of hotspot meshes
    let showDropdown = false;  // Toggle dropdown visibility

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

    function toggleDropdown() {
        showDropdown = !showDropdown;
    }

    function selectPanorama(index) {
        activePanoramaIndex = index;
        transitionToPanorama(index);  // Handle transition to the selected panorama
        showDropdown = false;  // Hide dropdown after selection
    }

    function transitionToPanorama(panoramaIndex) {
        // Transition logic (moved out of onMount)
        activeSphere.material.opacity = 0;  // Fade out old panorama
        setTimeout(() => {
            activeSphere.material.map = textures[panoramaIndex];  // Use preloaded texture
            activeSphere.material.opacity = 1;  // Fade in new panorama
            activePanoramaIndex = panoramaIndex;
            updateHotspots(scene);  // Update hotspots
        }, 500);  // Smooth transition
    }

    // Camera starting positions and rotations for each panorama
    let cameraPositions = [
        { position: new THREE.Vector3(-2.7, -0.5, 0.1), rotation: { x: 0, y: Math.PI / 2, z: 0 } },
        { position: new THREE.Vector3(-3, 0, 0.1), rotation: { x: 0, y: Math.PI / 4, z: 0 } },
        { position: new THREE.Vector3(-1, 0, 0.1), rotation: { x: 0, y: -Math.PI / 4, z: 0 } },
        { position: new THREE.Vector3(-2.7, 0, 0.1), rotation: { x: -Math.PI / 4, y: 0, z: 0 } },
        { position: new THREE.Vector3(-2, 0, 0.1), rotation: { x: Math.PI / 4, y: 0, z: 0 } },
        { position: new THREE.Vector3(-3, 0, 0.1), rotation: { x: 0, y: Math.PI / 4, z: 0 } }
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
        3: [getHotspot(200, 3.0, Math.PI / 2, 2),
        getHotspot(200, 1.75, Math.PI / 2, 5)
        ],
        
        4: [getHotspot(200, 4.5, Math.PI / 2, 2),
            getHotspot(200, 0, Math.PI / 2, 8)
        ],

        5: [getHotspot(200, 4.5, Math.PI / 2, 6)
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
        
        // Initialize camera position and rotation for the first panorama
        camera.position.copy(cameraPositions[activePanoramaIndex].position);
        camera.rotation.set(
            cameraPositions[activePanoramaIndex].rotation.x,
            cameraPositions[activePanoramaIndex].rotation.y,
            cameraPositions[activePanoramaIndex].rotation.z
        );

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
        controls.enableZoom = false;

        // Create initial hotspots (for the first panorama)
        createHotspots(scene);

        // Animation loop
        const animate = function () {
            requestAnimationFrame(animate);
            controls.update();
            
            // Make each hotspot face the camera
            hotspotMeshes.forEach(hotspotMesh => {
                hotspotMesh.lookAt(camera.position);
            });

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
            
            // Update camera position and rotation
            camera.position.copy(cameraPositions[panoramaIndex].position);
            camera.rotation.set(
                cameraPositions[panoramaIndex].rotation.x,
                cameraPositions[panoramaIndex].rotation.y,
                cameraPositions[panoramaIndex].rotation.z
            );

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
            const loader = new THREE.TextureLoader();
            const texture = loader.load('/arrow.png');  // Replace with your image path

            let hotspots = hotspotsByPanorama[activePanoramaIndex];
            hotspotMeshes = []; // Clear previous hotspots

            hotspots.forEach((hotspot) => {
                const hotspotMesh = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), new THREE.MeshBasicMaterial({
                    map: texture,
                    transparent: true,
                    opacity: 1,
                    side: THREE.DoubleSide
                }));

                hotspotMesh.position.set(hotspot.position.x, hotspot.position.y, hotspot.position.z);
                hotspotMesh.isHotspot = true;
                hotspotMesh.targetPanoramaIndex = hotspot.targetPanoramaIndex;

                // Make the hotspot face the camera
                hotspotMesh.lookAt(camera.position);

                // Add the hotspot to the scene and keep track of it
                scene.add(hotspotMesh);
                hotspotMeshes.push(hotspotMesh);
            });
        }

        function updateHotspots(scene) {
            // Remove existing hotspots
            hotspotMeshes.forEach(hotspotMesh => scene.remove(hotspotMesh));
            hotspotMeshes = []; // Clear the array
            // Create new hotspots for the current panorama
            createHotspots(scene);
        }
    });
</script>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        overflow: hidden;
        height: 100vh;
    }

    #container {
        display: flex;             
        justify-content: center;   
        align-items: center;       
        width: 100vw;              
        height: 100vh;             
        background-color: black;   
        position: relative;        
    }

    .dropdown {
        position: absolute;
        top: 10px;
        left: 10px;
        z-index: 1000;
        cursor: pointer;
        color: white;
        background-color: rgba(0, 0, 0, 0.7);
        border: 1px solid white;
        border-radius: 5px;
        padding: 5px;
        user-select: none;
    }

    .dropdown-menu {
    display: block;
    position: absolute;
    top: 30px;
    left: 0;
    z-index: 1001;
    background-color: rgba(0, 0, 0, 0.8);
    border: 1px solid white;
    border-radius: 5px;
    width: 200px;
    max-height: 200px; /* Set a max height for the dropdown */
    overflow-y: auto;  /* Enable vertical scrolling */
}


    .dropdown-item {
        padding: 10px;
        cursor: pointer;
        color: white;
        border-bottom: 1px solid white;
    }

    .dropdown-item:hover {
        background-color: rgba(255, 255, 255, 0.1);
    }
</style>

<div id="container" bind:this={container}></div>

<!-- Custom Dropdown -->
<div class="dropdown" on:click={toggleDropdown}>
    Select Panorama
    {#if showDropdown}
    <div class="dropdown-menu">
        {#each panoramaTextures as texture, index}
            <div class="dropdown-item" on:click={() => selectPanorama(index)}>
                {texture.split('/').pop().replace('.jpg', '')}
            </div>
        {/each}
    </div>
    {/if}
</div>