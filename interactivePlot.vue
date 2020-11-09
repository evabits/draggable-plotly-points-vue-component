<template>
<main>
    <div id="controls">
        <div>
            <p>Interpolation method</p>
            <select id="methodDropdown">
                <option value="Linear">Linear</option>
                <option value="FiniteDifference">Cubic (finite difference)</option>
                <option value="Cardinal">Cubic (cardinal)</option>
                <option value="FritschCarlson">Cubic monotonic (Fritsch-Carlson)</option>
                <option value="FritschButland" selected="selected">Cubic monotonic (Fritsch-Butland)</option>
                <option value="Steffen">Cubic monotonic (Steffen)</option>
            </select>
        </div>
        <div>
            <p>Tension (for cubic-cardinal only)</p>
            <input id="tensionSlider" type="range" min="0" max="1" value="0.5" step="0.1">
            <div id="tensionbox">0.5</div>
        </div>
        <div>
            <p>Test data</p>
            <select id="testdataDropdown">
                <option value="Akima" selected="selected">Akima dataset</option>
                <option value="Hussain1">Hussain dataset 1</option>
            </select>
        </div>
    </div>
    <div id="figurecontainer">
        <div id="spawntext">&darr; Drag this handle to the curve to add another breakpoint. Drag a handle back to the left
                to delete it. <br><span>Endpoints can only be moved vertically, so they can't be deleted.</span></div>
        <svg id="trash" width="50" height="50" viewBox="0 0 1792 1792" xmlns="http://www.w3.org/2000/svg"><path d="M704 736v576q0 14-9 23t-23 9h-64q-14 0-23-9t-9-23v-576q0-14 9-23t23-9h64q14 0 23 9t9 23zm256 0v576q0 14-9 23t-23 9h-64q-14 0-23-9t-9-23v-576q0-14 9-23t23-9h64q14 0 23 9t9 23zm256 0v576q0 14-9 23t-23 9h-64q-14 0-23-9t-9-23v-576q0-14 9-23t23-9h64q14 0 23 9t9 23zm128 724v-948h-896v948q0 22 7 40.5t14.5 27 10.5 8.5h832q3 0 10.5-8.5t14.5-27 7-40.5zm-672-1076h448l-48-117q-7-9-17-11h-317q-10 2-17 11zm928 32v64q0 14-9 23t-23 9h-96v948q0 83-47 143.5t-113 60.5h-832q-66 0-113-58.5t-47-141.5v-952h-96q-14 0-23-9t-9-23v-64q0-14 9-23t23-9h309l70-167q15-37 54-63t79-26h320q40 0 79 26t54 63l70 167h309q14 0 23 9t9 23z"/></svg>
    </div>
</main>
</template>

<script>
import Plotly from 'plotly.js'

export default {
    name: 'interactive-plot',
    data: function () {
        return {
            testdata: {
                Akima: {
                    x: [0, 2, 3, 5, 6, 8, 9, 11, 12, 14, 15],
                    y: [10, 10, 10, 10, 10, 10, 10.5, 15, 50, 60, 85],
                    range: {x: [-1, 16], y: [0, 90]}
                },
                Hussain1: {
                    x: [-9, -8, -4, 0, 4, 8, 9],
                    y: [7, 5, 3.5, 3.25, 3.5, 5, 7],
                    range: {x: [-11, 10], y: [3, 8]}
                },
            },
            figurecontainer: Object,
            layout: {
                autosize: true,
                showlegend: false,
                margin: {
                    t: 20,
                    r: 10,
                    b: 30,
                    l: 30,
                    pad: 0
                },
                xaxis: {
                    range: [0, 8],
                    fixedrange: false,
                    layer: 'below traces'
                },
                yaxis: {
                    range: [-10, 51],
                    fixedrange: false,
                    layer: 'below traces'
                },
                font: {size: 16}
            },
            interpolatedline: {
                x: [1, 8],
                y: [1, 40], 
                type: 'scatter',
                mode: 'lines',
                hoverinfo: 'none'
            },
            breakpoints: {
                x: [1, 8],
                y: [5, 30],
                type: 'scatter',
                cliponaxis: false,
                mode: 'markers',
                marker: {
                    size: 20,
                    symbol: "circle-open-dot",
                    color: '#b00',
                    line: {
                        width: 2
                    }
                },
                hoverinfo: 'none'
            },
            pointscontainer: Object, //figurecontainer.querySelector(".scatterlayer .trace:last-of-type .points"),
            points: Object, //pointscontainer.getElementsByTagName("path"),
            trash: Object,
            xspawn: 50, 
            yspawn: 50,      // pixel coordinates of the spawn handle
            m: [],
            n: [],
            delta: [],
            handles: [],
            interpolationmethod: 'FritschButland', // Linear, FiniteDifference, Cardinal, FritschCarlson, FritschButland, Steffen
            interpolationtension: 0.5,             // [0,1], only used for Cardinal splines
            firstx: [],         // Position of the leftmost breakpoint. Drag a handle beyond this to delete it.
            tensionSlider: Object,
            tensionbox: Object,
            methodDropdown: Object,
            testdataDropdown: Object,
        }
    },
    mounted() {        
        this.figurecontainer = document.getElementById("figurecontainer")
        this.trash = document.getElementById("trash")
        Plotly.plot(this.figurecontainer, [
            this.interpolatedline, 
            this.breakpoints,
        ], this.layout, {staticPlot: true})
        console.log( 'figurecontainer: ', this.figurecontainer )
        //this.pointscontainer = this.figurecontainer.querySelector(".scatterlayer .trace:last-of-type .points")
        this.pointscontainer = this.figurecontainer.querySelector(".scatterlayer .trace:nth-of-type(2n) .points")
        console.log( 'pointscontainer: ', this.pointscontainer )
        this.points = this.pointscontainer.getElementsByTagName("path")
        this.putOutTheTrash();
       
        this.methodDropdown   = document.getElementById("methodDropdown");
        this.tensionSlider    = document.getElementById("tensionSlider");
        this.tensionbox       = document.getElementById("tensionbox");
        this.testdataDropdown = document.getElementById("testdataDropdown");
        this.methodDropdown.selectedIndex = 4;
        this.tensionSlider.value = 0.5;
        this.testdataDropdown.selectedIndex = 1;
        console.log( "tensionSlider: ",    this.tensionSlider )
        console.log( "testdataDropdown: ", this.testdataDropdown )
        this.methodDropdown.onchange = function() {
            this.interpolationmethod = this.methodDropdown.options[this.methodDropdown.selectedIndex].value;
            this.updateFigure();
        }.bind(this)

        this.tensionSlider.oninput = function() {
            console.log( this )
            console.log( "tensionSlider: ", this.tensionSlider )
            this.interpolationtension = this.tensionSlider.value;
            this.tensionbox.innerHTML = this.tensionSlider.value;
            this.updateFigure();
        }.bind(this)

        this.testdataDropdown.onchange = function() {
            console.log( this.testdataDropdown )
            var datasource = this.testdataDropdown.options[this.testdataDropdown.selectedIndex].value;
            Plotly.relayout(this.figurecontainer, {
                'xaxis.range': this.testdata[datasource].range.x,
                'yaxis.range': this.testdata[datasource].range.y
            })
            var type;
            this.firstx = this.testdata[datasource].x[0];
            this.handles = [];
            this.addHandle('spawn');
            for (var i=0, len=this.testdata[datasource].x.length; i<len; i++) {
                if (datasource == "endpointsOnly") {
                    type = "endpoint";
                } else if (datasource == "hiddenDemo") {
                    type = i <= 1 ? "hidden" : i == len-1 ? "endpoint" : "normal";
                } else {
                    type = i == 0 || i == len-1 ? "endpoint" : "normal";
                }
                this.addHandle(type, this.testdata[datasource].x[i], this.testdata[datasource].y[i]);
            }
            this.updateFigure();
            this.updatePointHandles();
            this.startDragBehavior();
        }.bind(this)

        // load the default test data
        this.testdataDropdown.onchange();
    },
    methods: {
        interpolateCubicHermite: function(xeval, xbp, ybp, method, tension) {
            // first we need to determine tangents (m)
            this.n = xbp.length;
            var obj = this.calcTangents(xbp, ybp, method, tension);
            this.m = obj.m;          // length n
            this.delta = obj.delta;  // length n-1
            var c = new Array(this.n-1);
            var d = new Array(this.n-1);
            for (let k=0; k < this.n-1; k++) {
                if (this.interpolationmethod.toLowerCase() == 'linear') {
                    this.m[k] = this.delta[k];
                    c[k] = d[k] = 0;
                    continue;
                }
                let xdiff = xbp[k+1] - xbp[k];
                c[k] = (3*this.delta[k] - 2*this.m[k] - this.m[k+1]) / xdiff;
                d[k] = (this.m[k] + this.m[k+1] - 2*this.delta[k]) / xdiff / xdiff;
            }

            var len = xeval.length;
            var f = new Array(len);
            var k = 0;
            for (var i=0; i < len; i++) {
                var x = xeval[i];
                if (x < xbp[0] || x > xbp[this.n-1]) {
                    throw "interpolateCubicHermite: x value " + x + " outside breakpoint range [" + xbp[0] + ", " + xbp[this.n-1] + "]";
                }
                while (k < this.n-1 && x > xbp[k+1]) {
                    k++;
                }
                let xdiff = x - xbp[k];
                f[i] = ybp[k] + this.m[k]*xdiff + c[k]*xdiff*xdiff + d[k]*xdiff*xdiff*xdiff; 
            }
            return f;
        },
        calcTangents: function (x, y, method, tension) {
            method = typeof method === 'undefined' ? 'fritschbutland' : method.toLowerCase();
            var n = x.length;
            var delta = new Array(n-1);
            var m = new Array(n);
            for (var k=0; k < n-1; k++) {
                var deltak = (y[k+1] - y[k]) / (x[k+1] - x[k]);
                delta[k] = deltak;
                if (k == 0) {   // left endpoint, same for all methods
                    m[k] = deltak;
                } else if (method == 'cardinal') {
                    m[k] = (1 - tension) * (y[k+1] - y[k-1]) / (x[k+1] - x[k-1]);
                } else if (method == 'fritschbutland') {
                    var alpha = (1 + (x[k+1] - x[k]) / (x[k+1] - x[k-1])) / 3;  // Not the same alpha as below.
                    m[k] = delta[k-1] * deltak <= 0  ?  0 : delta[k-1] * deltak / (alpha*deltak + (1-alpha)*delta[k-1]);
                } else if (method == 'fritschcarlson') {
                    // If any consecutive secant lines change sign (i.e. curve changes direction), initialize the tangent to zero.
                    // This is needed to make the interpolation monotonic. Otherwise set tangent to the average of the secants.
                    m[k] = delta[k-1] * deltak < 0  ?  0 : (delta[k-1] + deltak) / 2;
                } else if (method == 'steffen') {
                    var p = ((x[k+1] - x[k]) * delta[k-1] + (x[k] - x[k-1]) * deltak) / (x[k+1] - x[k-1]);
                    m[k] = (Math.sign(delta[k-1]) + Math.sign(deltak)) * 
                                        Math.min(Math.abs(delta[k-1]), Math.abs(deltak), 0.5*Math.abs(p));
                } else {    // FiniteDifference
                    m[k] = (delta[k-1] + deltak) / 2;               
                }
            }
            m[n-1] = delta[n-2];
            if (method != 'fritschcarlson') {
                return {m: m, delta: delta};
            }

            /*
            Fritsch & Carlson derived necessary and sufficient conditions for monotonicity in their 1980 paper. Splines will be
            monotonic if all tangents are in a certain region of the alpha-beta plane, with alpha and beta as defined below.
            A robust choice is to put alpha & beta within a circle around origo with radius 3. The FritschCarlson algorithm
            makes simple initial estimates of tangents and then does another pass over data points to move any outlier tangents
            into the monotonic region. FritschButland & Steffen algorithms make more elaborate first estimates of tangents that
            are guaranteed to lie in the monotonic region, so no second pass is necessary. */
            
            // Second pass of FritschCarlson: adjust any non-monotonic tangents.
            for (let k=0; k < n-1; k++) {
                let deltak = delta[k];
                if (deltak == 0) {
                    m[k] = 0;
                    m[k+1] = 0;
                    continue;
                }
                let alpha = m[k] / deltak;
                let beta = m[k+1] / deltak;
                let tau = 3 / Math.sqrt(Math.pow(alpha,2) + Math.pow(beta,2));
                if (tau < 1) {      // if we're outside the circle with radius 3 then move onto the circle
                    m[k] = tau * alpha * deltak;
                    m[k+1] = tau * beta * deltak;
                }
            }
            return {m: m, delta: delta};
        },
        clamp: function (x, lower, upper) {
            return Math.max(lower, Math.min(x, upper));
        },
        linspace: function (start, end, n) {
            n = typeof n === "undefined" ? 500 : n;
            if (n <= 0) return [];
            var arr = Array(n-1);
            for (var i=0; i<=n-1; i++) {
                arr[i] = ((n-1-i)*start + i*end) / (n-1);
            }
            return arr;
        },
        sortHandles: function () {
            var len = this.handles.length;
            this.handles.sort(function(a,b) {
                return a.x-b.x;
            });
            var x = [], y = [], xvis = [], yvis = [], xmin = Infinity, xmax = -Infinity;
            for (var i=0; i < len; i++) {
                if (this.handles[i].type != 'spawn') {
                    x.push(this.handles[i].x);
                    y.push(this.handles[i].y);
                    xmin = this.handles[i].x < xmin ? this.handles[i].x : xmin;
                    xmax = this.handles[i].x > xmax ? this.handles[i].x : xmax;
                }
                if (this.handles[i].type != 'hidden') {
                    xvis.push(this.handles[i].x);
                    yvis.push(this.handles[i].y);
                }
            }
            return {x: x, y: y, xvis: xvis, yvis: yvis, xmin: xmin, xmax: xmax};
        },
        updateFigure: function () {
            var sortedhandles = this.sortHandles();
            var xx = this.linspace(sortedhandles.xmin, sortedhandles.xmax, 100);
            var yy = this.interpolateCubicHermite(xx, sortedhandles.x, sortedhandles.y, this.interpolationmethod, this.interpolationtension);
            Plotly.restyle( this.figurecontainer, {
                'x': [xx, sortedhandles.xvis], 
                'y': [yy, sortedhandles.yvis]}, 
                [0,1] 
            );
        },
        updatePointHandles: function () {
            for (var i=0, p=0, len=this.handles.length; i<len; i++) {
                if (this.handles[i].type != 'hidden') {
                    console.log( 'updatePointHandles ', i, p, this.points)
                    this.points[p++]['handle'] = this.handles[i];
                }
            }
        },
        destroyHandle: function (handle) {
            var i = this.handles.indexOf(handle);
            this.handles.splice(i,1);
            this.updateFigure();
        },
        poofHandle: function (handle) {
            Plotly.d3.select(this.points[0]).transition().duration(500)
                .attr("transform", "translate(" + this.xspawn + "," + this.yspawn + ") scale(0)")
                .each("end", function() {
                    this.destroyHandle(handle);
                });
        },
        addHandle: function (type, x, y) {
            if (type == 'spawn') {
                x = this.figurecontainer._fullLayout.xaxis.p2l(this.xspawn);
                y = this.figurecontainer._fullLayout.yaxis.p2l(this.yspawn);
            }
            var newhandle = {
                x: x,
                y: y,
                type: type
            };
            this.handles.push(newhandle);
            return newhandle;
        },
        startDragBehavior: function () {
            var d3 = Plotly.d3;
            var drag = d3.behavior.drag();
            console.log('startDragBehavior', this )
            var that = this;
            drag.origin( function() {
                var transform = d3.select(this).attr("transform");
                var translate = transform.substring(10, transform.length-1).split(/,| /);
                return {x: translate[0], y: translate[1]};
            }).bind(that);
            drag.on("dragstart", function() {
                if (this.handle.type != 'spawn') {
                    console.log( 'dragstart: ', this )
                    //console.log( 'dragstart: ', that )
                    that.trash.setAttribute("display", "inline");
                    that.trash.style.fill = "rgba(0,0,0,.2)";
                    that.destroyHandle(that.points[0].handle);
                }
            }).bind(that);
            drag.on("drag", function() {
                var xmouse = d3.event.x, ymouse = d3.event.y;
                d3.select(this).attr("transform", "translate(" + [xmouse, ymouse] + ")");
                var xaxis = that.figurecontainer._fullLayout.xaxis;
                var yaxis = that.figurecontainer._fullLayout.yaxis;
                var handle = this.handle;
                if (handle.type != 'endpoint') handle.x = that.clamp(xaxis.p2l(xmouse), xaxis.range[0], xaxis.range[1] - 1e-9);
                if (handle.type == 'spawn' && handle.x > that.handles[1].x) {
                    that.trash.setAttribute("display", "inline");
                    that.trash.style.fill = "rgba(0,0,0,.2)";
                    handle.type = 'normal';
                }
                handle.y = that.clamp(yaxis.p2l(ymouse), yaxis.range[0], yaxis.range[1]);
                if (handle.x < that.firstx) {    // release from the interpolation if dragged beyond the leftmost breakpoint
                    handle.type = 'spawn';
                    that.trash.style.fill = "#a00";              
                }
                that.updateFigure();
            }).bind(that);
            drag.on("dragend", function() {
                if (this.handle.x < that.firstx) that.destroyHandle(this.handle);
                that.addHandle('spawn');
                that.updateFigure();
                that.updatePointHandles();
                that.trash.setAttribute("display", "none");
                d3.select(".scatterlayer .trace:nth-of-type(2n) .points path:last-of-type").call(drag, {that: this});    
            }).bind(that);
            d3.selectAll(".scatterlayer .trace:nth-of-type(2n) .points path").call(drag, {that: this});
        },
        putOutTheTrash: function () {
            console.log( 'trash', this.trash )
            var trashsize = this.trash.getAttribute("width");
            this.pointscontainer.parentNode.insertBefore(this.trash, this.pointscontainer);
            this.trash.setAttribute("transform", "translate(" + (this.xspawn - trashsize/2) + "," + (this.yspawn - trashsize/2 + 5) + ")");
            this.trash.setAttribute("display", "none");
        }
    }
}
</script>

<style>
    #figurecontainer {
        margin: 0px;
        width: 960px;
        height: 800px;
        -webkit-touch-callout: none;
        -webkit-user-select: none;
        -khtml-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
    }
    #spawntext {
        display: inline-block;
        position: relative;
        top: -775px;
        left: 76px;
    }
    #spawntext span {
        position: relative;
        left: 50px;
    }
    #controls div {
        display: inline-block;
        margin: 20px 40px 0 40px;
        vertical-align: top;
    }
    #controls p {
        margin: 0;
    }
    #tensionbox {
        margin: 0 !important;
    }
    /* This is necessary to make Plotly points selectable. */
    .scatterlayer .trace:nth-child(2) path {
        pointer-events: all;
    }
</style>