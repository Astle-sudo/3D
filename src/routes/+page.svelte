<script>
// @ts-nocheck
	import P5 from 'p5-svelte';
    import { create, all, string } from 'mathjs';
    import PlotBlock from '../components/plot.svelte';
    import {Plot} from '../lib/store'
    import "../app.css";

    const math = create(all);
    let cubeSize = 120;
    let cameraDist = 2.5;
    let rotationSpeed = 0.06;
    let points = [],newPoints = [];
    let angle = 0;
    let previous,origin,movement,Current;
    let RotateAxisX = 0,RotateAxisY = 0;
    let R,G,B;

    function fX () {
        $Plot = [...$Plot,{
            str: 'X',
            IDnumber: $Plot.length + 1,
            Expression: null
        }]
    }
    function fY () {
        $Plot = [...$Plot,{
            str: 'Y',
            IDnumber: $Plot.length + 1,
            Expression: null
        }]
    }
    function fZ () {
        $Plot = [...$Plot,{
            str: 'Z',
            IDnumber: $Plot.length + 1,
            Expression: null
        }]
    }

	// @ts-ignore
	const sketch = (p5) => {

        class Matrix {
            constructor (arr) {
                this.arr = arr;
            }
            Copy() {
                let copyArr = this.arr.map((row) => [...row]);
                return new Matrix (copyArr);
            }
            add (matrix) {
                let dummyMatrix = this.Copy();
                for (let i = 0; i < this.arr.length; i++) {
                    for (let j = 0; j < this.arr[0].length; j++) {
                        dummyMatrix.arr[i][j] += matrix.arr[i][j];
                    }
                }
                return dummyMatrix;
            }
            sub (matrix) {
                let dummyMatrix = this.Copy();
                for (let i = 0; i < this.arr.length; i++) {
                    for (let j = 0; j < this.arr[0].length; j++) {
                        dummyMatrix.arr[i][j] -= matrix.arr[i][j];
                    }
                }
                return dummyMatrix;
            }
            mul (matrix) {
                let dummyMatrix = this.Copy();
                let m = this.arr[0].length;
                let n = this.arr.length;
                let p = matrix.arr[0].length;
                let ans = CreateMatrix(n,p);
                for (let i = 0; i < n; i++) {
                    for (let j = 0; j < p; j++) {
                        let sum = 0;
                        for (let k = 0; k < m; k++) {
                            sum = sum + (dummyMatrix.arr[i][k] * matrix.arr[k][j]);
                        }
                        ans[i][j] = sum;
                    }
                }
                return new Matrix(ans);
            }
            scalar (number) {
                for (let i = 0; i < this.arr.length; i++) {
                    for (let j = 0; j < this.arr[0].length; j++) {
                        this.arr[i][j] *= number;
                    }
                }
                return new Matrix (this.arr);
            }
            T () {
                let dummyMatrix = this.Copy();
                let ans = CreateMatrix (this.arr[0].length,this.arr.length);
                for (let i = 0; i < this.arr[0].length; i++) {
                    for (let j = 0; j < this.arr.length; j++) {
                        ans[i][j] = dummyMatrix.arr[j][i];
                    }
                }
                return new Matrix(ans);
            }
            Dot (matrix) {
                let dummyMatrix = this.Copy();
                for (let i = 0; i < this.arr.length; i++) {
                    for (let j = 0; j < this.arr[0].length; j++) {
                        dummyMatrix.arr[i][j] *= matrix.arr[i][j];
                    }
                }
            return dummyMatrix;
            }
        }
        class Quarternions {
        constructor (w,x,y,z) {
            this.w = w;
            this.x = x;
            this.y = y;
            this.z = z;
        }
        mul (Q) {
            let ans = new Quarternions (0,0,0,0);
            ans.w += (this.w * Q.w - this.x * Q.x - this.y * Q.y - this.z * Q.z);
            ans.x += (this.w * Q.x + this.x * Q.w - this.y * Q.z + this.z * Q.y);
            ans.y += (this.w * Q.y + this.x * Q.z + this.y * Q.w - this.z * Q.x);
            ans.z += (this.w * Q.z - this.x * Q.y + this.y * Q.x + this.z * Q.w);
            return ans;
        }
        inverse () {
            let ans = new Quarternions (this.w,-this.x,-this.y,-this.z);
            return ans;
        }
        }
        function CreateMatrix (rows,cols) {
            let matrix = [];
            for (let i = 0; i < rows; i++) {
                matrix.push([]);
                for (let j = 0; j < cols; j++) {
                    matrix[i].push(0);
                }
            }
            return matrix;
        }
        function DrawCube (newPoints) {
            function connectLines (index1,index2,points) {
                p5.push()
                p5.stroke(80);
                p5.strokeWeight(1);
                p5.line(points[index1][0],points[index1][1],points[index2][0],points[index2][1]);
                p5.pop()
            }
            for (let i = 0; i < 4; i++) {
                connectLines (i,(i+1)%4,newPoints);
                connectLines (i+4,((i+1)%4)+4,newPoints);
                connectLines (i,i+4,newPoints);
            }
        }
        function AngleAxisToQuarternion (angle,x,y,z) {
            if (x == 0 && y == 0 && z == 0) {
                angle = 0;
            }
            let axis = p5.createVector(x,y,z);
            let normAxis = axis.normalize();
            return new Quarternions(p5.cos(angle/2),normAxis.x*p5.sin(angle/2),normAxis.y*p5.sin(angle/2),normAxis.z*p5.sin(angle/2));
        }
        function RotationMatrix (Q) {
            let q0 = Q.w;
            let q1 = Q.x;
            let q2 = Q.y;
            let q3 = Q.z;
            return new Matrix ([[q0**2+q1**2-q2**2-q3**2, 2*(q1*q2-q0*q3), 2*(q1*q3+q2*q0)],
                                [2*(q1*q2+q0*q3), q0**2-q1**2+q2**2-q3**2, 2*(q2*q3-q0*q1)],
                                [2*(q1*q3-q0*q2), 2*(q2*q3+q0*q1), q0**2-q1**2-q2**2+q3**2]])
        }
        function Projection (Point,RotationQuarternion) {
            let p = Point;
            let q = RotationQuarternion;
            let q_i = q.inverse();
            let pq = p.mul(q);
            let qp = q.mul(p)
  
            return q_i.mul(pq);
        }
        function ApplyProjection (Q_point,distance) {
            let z = 1 / (distance - Q_point.z*0.01);
            let pointMatrix = new Matrix ([[Q_point.x],[Q_point.y],[Q_point.z]]);
            let projectionMatrix = new Matrix ([[z,0,0],
                                                [0,z,0]]);
            let projectedPoint = projectionMatrix.mul(pointMatrix);
            return projectedPoint.T().arr[0]; 
        }
        function ToQuarternion (Point) {
            return new Quarternions (0,Point.arr[0][0],Point.arr[0][1],Point.arr[0][2]);
        }
        function threeD_twoD (threeD_point,angle,RotateAxisX,RotateAxisY,cameraDistance) {
            let Q = ToQuarternion (threeD_point);
            let rotationQuarternion = AngleAxisToQuarternion (angle,RotateAxisX,RotateAxisY,0);
            let projectedPoint = Projection(Q,rotationQuarternion);
            return ApplyProjection(projectedPoint,cameraDistance)
        }
        function RotateCube (angle,cameraDistance,RotateAxisX,RotateAxisY) {
            newPoints = [];
            for (let p of points) {    
                newPoints.push(threeD_twoD(p,angle,RotateAxisX,RotateAxisY,cameraDistance));
            }
            DrawCube(newPoints);
        }
        function generatePlot (R,G,B,str,expression) {    
            if (str == 'X') {
                for (let i = -2; i < 2; i += 0.1) {
                    for (let j = -2; j < 2; j+= 0.1) {
                        let scope = {
                            y: i,
                            z: j
                        }
                        p5.push();
                        let p = new Matrix ([[math.evaluate(expression,scope),i,j]]);
                        let newP = threeD_twoD(p,angle,RotateAxisX,RotateAxisY,cameraDist);
                        p5.stroke(R,G,B);
                        p5.strokeWeight(4);
                        p5.point(newP[0] * 100,newP[1] * 100);
                        p5.pop();
                    }
                }
            }
            if (str == 'Y') {
                for (let i = -2; i < 2; i += 0.1) {
                    for (let j = -2; j < 2; j+= 0.1) {
                        let scope = {
                            x: i,
                            z: j
                        }
                        p5.push();
                        let p = new Matrix ([[i,math.evaluate(expression,scope),j]]);
                        let newP = threeD_twoD(p,angle,RotateAxisX,RotateAxisY,cameraDist);
                        p5.stroke(R,G,B);
                        p5.strokeWeight(4);
                        p5.point(newP[0] * 100,newP[1] * 100);
                        p5.pop();
                    }
                }
            }
            if (str == 'Z') {
                for (let i = -2; i < 2; i += 0.1) {
                    for (let j = -2; j < 2; j+= 0.1) {
                        let scope = {
                            x: i,
                            y: j
                        }
                        p5.push();
                        let p = new Matrix ([[i,j,math.evaluate(expression,scope)]]);
                        let newP = threeD_twoD(p,angle,RotateAxisX,RotateAxisY,cameraDist);
                        p5.stroke(R,G,B);
                        p5.strokeWeight(4);
                        p5.point(newP[0] * 100,newP[1] * 100);
                        p5.pop();
                    }
                }
            }
        }

		p5.setup = () => {
			p5.createCanvas(850, 850);
  
            //Cube Points
            points.push(new Matrix ([[-1,-1,-1]]));
            points.push(new Matrix ([[1,-1,-1]]));
            points.push(new Matrix ([[1,1,-1]]));
            points.push(new Matrix ([[-1,1,-1]]));
            points.push(new Matrix ([[-1,-1,1]]));
            points.push(new Matrix ([[1,-1,1]]));
            points.push(new Matrix ([[1,1,1]]));
            points.push(new Matrix ([[-1,1,1]]));
  
            origin = new Matrix ([[0,0,0]]);
            RotateAxisX = 0;
            RotateAxisY = 0;
            R = 255;
            G = 255;
            B = 255;
  
            for (let p of points) {
                p.scalar(cubeSize);
                newPoints.push(p.arr[0]);
            }
		};
		p5.draw = () => {
			p5.background(0);
            p5.translate(p5.height/2,p5.width/2);
            
            for (let plots of $Plot) {
                if (plots.Expression != null) {
                    generatePlot(R,G,B,plots.str,plots.Expression);
                }
            }
  
            if (p5.mouseIsPressed) {
                Current = p5.createVector((p5.mouseX-(p5.height/2)),(p5.mouseY-(p5.width/2)));
                previous = p5.createVector((p5.pmouseX-(p5.height/2)),(p5.pmouseY-(p5.width/2)));
                movement = Current.sub(previous);
                RotateAxisX += -movement.y;
                RotateAxisY += movement.x;
            }
  
            //RotateCube (angle,cameraDist,RotateAxisX,RotateAxisY);
  
            p5.push();
            p5.stroke(150);
            p5.strokeWeight(1);

            let X = new Matrix([[200,0,0]]);
            let Y = new Matrix([[0,200,0]]);
            let Z = new Matrix([[0,0,200]]);
            let X_i = new Matrix([[-200,0,0]]);
            let Y_i = new Matrix([[0,-200,0]]);
            let Z_i = new Matrix([[0,0,-200]]);
            let P1 = threeD_twoD (X,angle,RotateAxisX,RotateAxisY,cameraDist);
            let P2 = threeD_twoD (Y,angle,RotateAxisX,RotateAxisY,cameraDist);
            let P3 = threeD_twoD (Z,angle,RotateAxisX,RotateAxisY,cameraDist);
            let P4 = threeD_twoD (X_i,angle,RotateAxisX,RotateAxisY,cameraDist);
            let P5 = threeD_twoD (Y_i,angle,RotateAxisX,RotateAxisY,cameraDist);
            let P6 = threeD_twoD (Z_i,angle,RotateAxisX,RotateAxisY,cameraDist);
            let O = threeD_twoD (origin,angle,RotateAxisX,RotateAxisY,cameraDist);

            p5.line(P1[0],P1[1],O[0],O[1]);
            p5.line(P2[0],P2[1],O[0],O[1]);
            p5.line(P3[0],P3[1],O[0],O[1]);
            p5.line(P4[0],P4[1],O[0],O[1]);
            p5.line(P5[0],P5[1],O[0],O[1]);
            p5.line(P6[0],P6[1],O[0],O[1]);
            p5.push();
            p5.stroke(255);
            p5.strokeWeight(4);
            p5.text(' X ',P1[0],P1[1]);
            p5.text(' Y ',P2[0],P2[1]);
            p5.text(' Z ',P3[0],P3[1]);            
            p5.pop();
            p5.pop()
  
            if (RotateAxisX > 20) {
                RotateAxisX = 20;
            }
            if (RotateAxisX < -20) {
                RotateAxisX = -20;
            }
            if (RotateAxisY > 20) {
                RotateAxisY = 20;
            }
            if (RotateAxisY < -20) {
                RotateAxisY = -20;
            }
            //p5.noLoop();
		};
        p5.mouseDragged = () => {
            angle += rotationSpeed;
        }
	};
</script>

<div class="addFunction">
    <h2>
        Add Function +
    </h2>
   <div class="buttonContainer">
    <button on:click={fX}>X = f (y,z)</button>
    <button on:click={fY}>Y = f (x,z)</button>
    <button on:click={fZ}>Z = f (x,y)</button>
   </div>
</div>

<div class="blockContainer">
    {#each $Plot as plotPoint}
        <PlotBlock str={plotPoint.str} ID={plotPoint.IDnumber}/>
    {/each}
</div>

<div class="drawing">
    <P5 {sketch} />
</div>

<style>
    h2 {
        margin-left: 15%;
        text-align: center;
    }
    .drawing {
        padding-left: 40.5vw;
    }
    .addFunction {
        scale: 1.2;
        position: absolute;
        font-weight: bold;
        font-size: large;
        top: 2.5%;
        left: 13%;
    }
    .addFunction:hover {
        cursor: pointer;
    }
    button {
        padding: 1%;
        font-size: small;
        color: black;
        border: 1px solid black;
    }
    button:hover {
        background-color: black;
        color: white;
    }
    .buttonContainer {
        width: 120%;
        display: flex;
        justify-content: space-between;
    }
    .blockContainer {
        max-height: 90%;
        width: 100%;
        position: absolute;
        top: 10%;
        left: 4%;
        overflow-y: scroll;
    }
</style>