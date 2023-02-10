<script>
    import {getContext, onMount} from "svelte"

    export let imageUrl

    const {styleable} = getContext("sdk")
    const component = getContext("component")
    let canvas;

    let infosPoints = [
        {
            "nom": "Aucune sélection",
            "image": ""
        },
        {
            "nom": "P : sondage pénétromètre dynamique",
            "image": "p.png",
            "diminutif": "P"
        },
        {
            "nom": "Pr : sondage pressiométrique",
            "image": "pr.png",
            "diminutif": "Pr"
        },
        {
            "nom": "Es : Evaluation SPT ou Tir sismique",
            "image": "es.png",
            "diminutif": "Es"
        },
        {
            "nom": "T : sondage tarière",
            "image": "t.png",
            "diminutif": "T"
        },
        {
            "nom": "SPM : sondage pelle mécanique ou reco",
            "image": "spm.png",
            "diminutif": "SPM"
        },
        {
            "nom": "Piezzo : relevé piézométrique avec niveau stabilisé",
            "image": "piezzo.png",
            "diminutif": "Piez"
        },
        {
            "nom": "Arbre repéré",
            "image": "arbre.png",
            "diminutif": "A"
        },
        {
            "nom": "Puit repéré",
            "image": "puit.png",
            "diminutif": "Puit"
        },
        {
            "nom": "Tracé libre",
            "image": "puit.png",
            "role": "trace"
        }
    ];
    let cameraOffset = {x: 0, y: 0};
    let cameraZoom = 1;
    let MAX_ZOOM = 5;
    let MIN_ZOOM = 0.1;
    let SCROLL_SENSITIVITY = 0.0005;
    let DRAG_SENSITIVITY = 0.05;
    let isDragging = false;
    let dragStart = {x: window.innerWidth / 2, y: window.innerHeight / 2};
    let initialPinchDistance = null;
    let lastZoom = cameraZoom;

    let taillePoint = {width: 30, height: 30};
    let distanceLegende = {x: 10, y: 0};

    let selectionPoint = 0;
    let traceLibre = false; //précise si on est en train de tracer ou pas
    let enregistrementTrace = false; //garde en mémoire s'il y a un déjà un enregistrement en cours
    let pointsEnregistres = [];
    let legendesEnregistres = [];
    let tracesEnregistres = [];
    let pointTouche = -1;
    let pointTrace = {
        width: 5,
        height: 5
    }

    let img = new Image();
    let ctx;
    onMount(() => {
        ctx = canvas.getContext('2d');
        img.src = imageUrl;
        img.crossOrigin = "anonymous";

        img.onload = () => {
            majPositionImage();
        }
        canvas.addEventListener('mousedown', (e) => onPointerDown(e));
        canvas.addEventListener('touchstart', (e) => handleTouch(e));
        canvas.addEventListener('mouseup', (e) => onPointerUp(e));
        canvas.addEventListener('touchend', (e) => handleTouch(e))
        canvas.addEventListener('mousemove', (e) => onPointerMove(e))
        canvas.addEventListener('touchmove', (e) => handleTouch(e))
        canvas.addEventListener('wheel', (e) => adjustZoom(e.deltaY * SCROLL_SENSITIVITY, null))
    });

    function majPositionImage() {
        let ratioCanvas = canvas.width / canvas.height;
        let ratioImg = img.width / img.height;
        ctx.canvas.width = img.width * cameraZoom;
        ctx.canvas.height = img.height * cameraZoom;

        ctx.fillStyle = "white";
        ctx.fillRect(0, 0, ctx.canvas.width, ctx.canvas.height);
        ctx.drawImage(img, cameraOffset.x, cameraOffset.y);

        let i =0;
        for(let point of pointsEnregistres){
            ctx.drawImage(point.image, point.x + cameraOffset.x, point.y + cameraOffset.y, taillePoint.width ,taillePoint.height);        // FILL THE CANVAS WITH THE IMAGE.

            //affichage de la légende
            //création de la légende
            let coord_x = point.x + cameraOffset.x +taillePoint.width;
            let coord_y = point.y + cameraOffset.y;
            ctx.font = '20px arial';
            ctx.fillStyle = "black";
            let texte = getNomPoint(i);
            ctx.fillText(texte,  coord_x, coord_y); //this.infosPoints[type].diminutif + this.getNumeroPoint()
            i++;
        }

        for(let trace of tracesEnregistres){
            for(let point of trace){

                let coord_x = point.x + cameraOffset.x;
                let coord_y = point.y + cameraOffset.y;

                ctx.fillStyle = "black";
                ctx.fillRect(coord_x, coord_y, pointTrace.width, pointTrace.height);
            }
        }

        ctx.translate(window.innerWidth / 2, window.innerHeight / 2)
        ctx.scale(cameraZoom, cameraZoom)
        ctx.translate(-window.innerWidth / 2 + cameraOffset.x, -window.innerHeight / 2 + cameraOffset.y)

    }

    function getEventLocation(e) {
        if (e.touches && e.touches.length === 1) {
            return {x: e.touches[0].clientX, y: e.touches[0].clientY}
        } else if (e.clientX && e.clientY) {
            return {x: e.clientX, y: e.clientY}
        } else return {x: '', y: ''}
    }

    function getMousePos(mouseEvent) {
        let rect = canvas.getBoundingClientRect();

        return {
            x: mouseEvent.clientX - rect.left,
            y: mouseEvent.clientY - rect.top
        };
    }

    function onPointerDown(e) {

        //réaction au tracé libre
        if (traceLibre && !enregistrementTrace) {
            enregistrementTrace = true;
            tracesEnregistres.push([]);
        }

        //console.log("onpointerdown");
        isDragging = true
        dragStart.x = getEventLocation(e).x / cameraZoom - cameraOffset.x; //*this.DRAG_SENSITIVITY
        dragStart.y = getEventLocation(e).y / cameraZoom - cameraOffset.y; //*this.DRAG_SENSITIVITY - this.cameraOffset.y
        majPositionImage();
    }

    function onPointerUp(e) {
        //console.log("onpointerup");
        if (enregistrementTrace) {
            selectionPoint = -1;
        }

        enregistrementTrace = false;
        traceLibre = false;
        isDragging = false;
        initialPinchDistance = null;
        lastZoom = cameraZoom;

        //Gestion d'un ajout de point
        if (selectionPoint > 0) {
            //console.log("ajout d'un point");
            let coord = calculCoordonnees(getMousePos(e).x, getMousePos(e).y);
            ajouterPoint(coord[0], coord[1], selectionPoint);
        } else if (clickSurPoint(getMousePos(e).x, getMousePos(e).y) >= 0) {
            let indice = clickSurPoint(getMousePos(e).x, getMousePos(e).y);
            //afficher la modal de modification
            pointTouche = indice;
            clickAfficherModal('modifier-point');
        }

        majPositionImage();
    }

    function clickSurPoint(x, y) {
        //console.log(x+" "+y);
        let coords = calculCoordonnees(x, y);
        x = coords[0];
        y = coords[1];
        let indice = 0;
        for (let point of pointsEnregistres) {
            let cx = point.x;
            let cy = point.y;
            if (cx - taillePoint.width / 2 <= x && x <= cx + taillePoint.width / 2) {
                if (cy - taillePoint.height / 2 <= y && y <= cy + taillePoint.height / 2) {

                    return indice;

                }
            }
            indice++;
        }
        return -1;
    }

    function calculCoordonnees(x, y, decalage) {
        if (decalage === undefined) {
            decalage = taillePoint.width / 2;
        }
        let rect = canvas.getBoundingClientRect();
        x = x * img.width / rect.width * cameraZoom - decalage - cameraOffset.x; ///this.ctx.canvas.width*this.img.width
        y = y * img.height / rect.height * cameraZoom - decalage - cameraOffset.y; /// this.ctx.canvas.height*this.img.height
        return [x, y];
    }

    function ajouterPointTrace(x, y) {
        let point = {
            x: x,
            y: y
        };
        //console.log(point);
        tracesEnregistres[tracesEnregistres.length - 1].push(point);
        majPositionImage();
    }

    function ajouterPoint(x, y, type) {
        let rect = canvas.getBoundingClientRect();

        //Ajout de l'image
        let image = new Image();
        image.src = "https://placekitten.com/640/360";//"assets/icones-ope/" + infosPoints[type].image;
        image.onload = () => {
            //this.canvas.nativeElement.width = img.width;
            //this.canvas.nativeElement.height = img.height;
            let point = {
                type: type,
                x: x,
                y: y,
                image: image
            };


            console.log(point);
            pointsEnregistres.push(point);
            selectionPoint = 0;
            majPositionImage();
        }
    }

    function onPointerMove(e) {
        //console.log("onpointermove");
        if (traceLibre && !enregistrementTrace) {
            enregistrementTrace = true;
            tracesEnregistres.push([]);
        }
        //gestion du trace
        if (enregistrementTrace) {
            //console.log(this.getMousePos(e).x + " "+ this.getMousePos(e).y);
            let coords = calculCoordonnees(getMousePos(e).x, getMousePos(e).y, 0);
            //console.log(coords);
            ajouterPointTrace(coords[0], coords[1]);
            majPositionImage();
        }

        // gestion du dragging
        else if (isDragging) {
            //console.log("dragging");
            cameraOffset.x = getEventLocation(e).x / cameraZoom - dragStart.x;
            cameraOffset.y = getEventLocation(e).y / cameraZoom - dragStart.y;

            majPositionImage();
        }
    }

    function handleTouch(e) {
        //console.log("handletouch");


        if (e.touches.length === 1) {
            if (e.type === 'touchstart') {
                console.log("touchstart");
                onPointerDown(e);
            } else if (e.type === 'touchend') {
                console.log("touchend");
                onPointerUp(e);
            } else if (e.type === 'touchmove') {
                onPointerMove(e);
            }
        } else if (e.type === "touchmove" && e.touches.length === 2) {
            isDragging = false
            handlePinch(e)
        }
        majPositionImage();
    }

    function handlePinch(e) {
        //console.log("handlepinch");
        e.preventDefault()

        let touch1 = {x: e.touches[0].clientX, y: e.touches[0].clientY}
        let touch2 = {x: e.touches[1].clientX, y: e.touches[1].clientY}

        // This is distance squared, but no need for an expensive sqrt as it's only used in ratio
        let currentDistance = (touch1.x - touch2.x) ** 2 + (touch1.y - touch2.y) ** 2

        if (initialPinchDistance == null) {
            initialPinchDistance = currentDistance
        } else {
            adjustZoom(null, initialPinchDistance / currentDistance)
        }
        majPositionImage();
    }

    function adjustZoom(zoomAmount, zoomFactor) {
        //console.log("adjustzoom");
        if (!isDragging) {
            if (zoomAmount) {
                cameraZoom += zoomAmount;
            } else if (zoomFactor) {
                //console.log(zoomFactor)
                cameraZoom = zoomFactor * lastZoom
            }

            cameraZoom = Math.min(cameraZoom, MAX_ZOOM)
            cameraZoom = Math.max(cameraZoom, MIN_ZOOM)

            //console.log(zoomAmount)
        }
        majPositionImage();
    }

    function afficherModal() {
        return ""//this.outilsService.afficherModal();
    }

    function afficherContenuModal(id) {
        return ""//this.outilsService.afficherContenuModal(id);
    }

    function clickAfficherModal(id) {
        return ""//this.outilsService.clickAfficherModal(id);
    }

    function annulerSuppressionPoint() {
        pointTouche = -1;
        clickAfficherModal('modifier-point');
    }

    function supprimerPoint(index) {
        pointsEnregistres.splice(index, 1);
        majPositionImage();
        clickAfficherModal('modifier-point');
    }

    function getNomPoint(index) {
        let point = pointsEnregistres[index];
        return infosPoints[point.type].diminutif + getNumeroPoint(point.type, index);
    }

    function getNumeroPoint(type, index) {
        //console.log(this.pointsEnregistres);

        let count = 1;
        for (let i = 0; i < index; i++) {
            let point = pointsEnregistres[i];
            if (point.type === type) {
                count++;
            }
        }
        //console.log(count);
        return count;
    }
    function choixPrelevement(index){
        console.log("touché");
        selectionPoint = index;
        if(infosPoints[index]['role'] !== undefined){
            if(infosPoints[index]['role'] === 'trace'){
                traceLibre=true;
            }
        }else{
            console.log("trace avant click"+traceLibre);
            traceLibre=false;
            console.log("trace après click"+traceLibre);
        }
    }

</script>

<div class="component" use:styleable={$component.styles}>
    <div class="icons">
        {#each infosPoints as icon, index}
            <div on:click={() => choixPrelevement(index)} class="container-icons">
                <div>
                    <img src="{imageUrl}" alt="icon" class="image" style="height:32px;width:32px;">
                </div>

                <div class="text-container">
                    <span>{icon['nom']}</span>
                </div>
            </div>
        {/each}
    </div>


    <canvas bind:this={canvas}/>
</div>

<style>
    canvas{
        border: 2px solid black;
    }
    .component{
        display: flex;
        flex-flow: row wrap;
        justify-content: space-between;
        width: 100%;
        height: 100%;
    }
    .icons{
        width: 20%;
        height: 70%;
    }
    .container-icons{
        display: flex;
        align-items: center;
        overflow: scroll;
    }
    .image{
        border-radius: 50%;
        width: 32px;
        height: 32px;
        object-fit: cover;
        overflow: hidden;
    }
    .text-container{
        color: black;
        font-size: 12px;
        font-weight: 600;
        line-height: 16px;
        overflow-wrap: break-word;
        margin-left: 8px;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>
