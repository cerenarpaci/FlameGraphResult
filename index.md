<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg version="1.1" width="1200" height="966" onload="init(evt)" viewBox="0 0 1200 966" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
<!-- Flame graph stack visualization. See https://github.com/brendangregg/FlameGraph for latest version, and http://www.brendangregg.com/flamegraphs.html for examples. -->
<!-- NOTES:  -->
<defs>
	<linearGradient id="background" y1="0" y2="1" x1="0" x2="0" >
		<stop stop-color="#eeeeee" offset="5%" />
		<stop stop-color="#eeeeb0" offset="95%" />
	</linearGradient>
</defs>
<style type="text/css">
	text { font-family:Verdana; font-size:12px; fill:rgb(0,0,0); }
	#search, #ignorecase { opacity:0.1; cursor:pointer; }
	#search:hover, #search.show, #ignorecase:hover, #ignorecase.show { opacity:1; }
	#subtitle { text-anchor:middle; font-color:rgb(160,160,160); }
	#title { text-anchor:middle; font-size:17px}
	#unzoom { cursor:pointer; }
	#frames > *:hover { stroke:black; stroke-width:0.5; cursor:pointer; }
	.hide { display:none; }
	.parent { opacity:0.5; }
</style>
<script type="text/ecmascript">
<![CDATA[
	"use strict";
	var details, searchbtn, unzoombtn, matchedtxt, svg, searching, currentSearchTerm, ignorecase, ignorecaseBtn;
	function init(evt) {
		details = document.getElementById("details").firstChild;
		searchbtn = document.getElementById("search");
		ignorecaseBtn = document.getElementById("ignorecase");
		unzoombtn = document.getElementById("unzoom");
		matchedtxt = document.getElementById("matched");
		svg = document.getElementsByTagName("svg")[0];
		searching = 0;
		currentSearchTerm = null;
	}

	window.addEventListener("click", function(e) {
		var target = find_group(e.target);
		if (target) {
			if (target.nodeName == "a") {
				if (e.ctrlKey === false) return;
				e.preventDefault();
			}
			if (target.classList.contains("parent")) unzoom();
			zoom(target);
		}
		else if (e.target.id == "unzoom") unzoom();
		else if (e.target.id == "search") search_prompt();
		else if (e.target.id == "ignorecase") toggle_ignorecase();
	}, false)

	// mouse-over for info
	// show
	window.addEventListener("mouseover", function(e) {
		var target = find_group(e.target);
		if (target) details.nodeValue = "Function: " + g_to_text(target);
	}, false)

	// clear
	window.addEventListener("mouseout", function(e) {
		var target = find_group(e.target);
		if (target) details.nodeValue = ' ';
	}, false)

	// ctrl-F for search
	window.addEventListener("keydown",function (e) {
		if (e.keyCode === 114 || (e.ctrlKey && e.keyCode === 70)) {
			e.preventDefault();
			search_prompt();
		}
	}, false)

	// ctrl-I to toggle case-sensitive search
	window.addEventListener("keydown",function (e) {
		if (e.ctrlKey && e.keyCode === 73) {
			e.preventDefault();
			toggle_ignorecase();
		}
	}, false)

	// functions
	function find_child(node, selector) {
		var children = node.querySelectorAll(selector);
		if (children.length) return children[0];
		return;
	}
	function find_group(node) {
		var parent = node.parentElement;
		if (!parent) return;
		if (parent.id == "frames") return node;
		return find_group(parent);
	}
	function orig_save(e, attr, val) {
		if (e.attributes["_orig_" + attr] != undefined) return;
		if (e.attributes[attr] == undefined) return;
		if (val == undefined) val = e.attributes[attr].value;
		e.setAttribute("_orig_" + attr, val);
	}
	function orig_load(e, attr) {
		if (e.attributes["_orig_"+attr] == undefined) return;
		e.attributes[attr].value = e.attributes["_orig_" + attr].value;
		e.removeAttribute("_orig_"+attr);
	}
	function g_to_text(e) {
		var text = find_child(e, "title").firstChild.nodeValue;
		return (text)
	}
	function g_to_func(e) {
		var func = g_to_text(e);
		// if there's any manipulation we want to do to the function
		// name before it's searched, do it here before returning.
		return (func);
	}
	function update_text(e) {
		var r = find_child(e, "rect");
		var t = find_child(e, "text");
		var w = parseFloat(r.attributes.width.value) -3;
		var txt = find_child(e, "title").textContent.replace(/\([^(]*\)$/,"");
		t.attributes.x.value = parseFloat(r.attributes.x.value) + 3;

		// Smaller than this size won't fit anything
		if (w < 2 * 12 * 0.59) {
			t.textContent = "";
			return;
		}

		t.textContent = txt;
		// Fit in full text width
		if (/^ *$/.test(txt) || t.getSubStringLength(0, txt.length) < w)
			return;

		for (var x = txt.length - 2; x > 0; x--) {
			if (t.getSubStringLength(0, x + 2) <= w) {
				t.textContent = txt.substring(0, x) + "..";
				return;
			}
		}
		t.textContent = "";
	}

	// zoom
	function zoom_reset(e) {
		if (e.attributes != undefined) {
			orig_load(e, "x");
			orig_load(e, "width");
		}
		if (e.childNodes == undefined) return;
		for (var i = 0, c = e.childNodes; i < c.length; i++) {
			zoom_reset(c[i]);
		}
	}
	function zoom_child(e, x, ratio) {
		if (e.attributes != undefined) {
			if (e.attributes.x != undefined) {
				orig_save(e, "x");
				e.attributes.x.value = (parseFloat(e.attributes.x.value) - x - 10) * ratio + 10;
				if (e.tagName == "text")
					e.attributes.x.value = find_child(e.parentNode, "rect[x]").attributes.x.value + 3;
			}
			if (e.attributes.width != undefined) {
				orig_save(e, "width");
				e.attributes.width.value = parseFloat(e.attributes.width.value) * ratio;
			}
		}

		if (e.childNodes == undefined) return;
		for (var i = 0, c = e.childNodes; i < c.length; i++) {
			zoom_child(c[i], x - 10, ratio);
		}
	}
	function zoom_parent(e) {
		if (e.attributes) {
			if (e.attributes.x != undefined) {
				orig_save(e, "x");
				e.attributes.x.value = 10;
			}
			if (e.attributes.width != undefined) {
				orig_save(e, "width");
				e.attributes.width.value = parseInt(svg.width.baseVal.value) - (10 * 2);
			}
		}
		if (e.childNodes == undefined) return;
		for (var i = 0, c = e.childNodes; i < c.length; i++) {
			zoom_parent(c[i]);
		}
	}
	function zoom(node) {
		var attr = find_child(node, "rect").attributes;
		var width = parseFloat(attr.width.value);
		var xmin = parseFloat(attr.x.value);
		var xmax = parseFloat(xmin + width);
		var ymin = parseFloat(attr.y.value);
		var ratio = (svg.width.baseVal.value - 2 * 10) / width;

		// XXX: Workaround for JavaScript float issues (fix me)
		var fudge = 0.0001;

		unzoombtn.classList.remove("hide");

		var el = document.getElementById("frames").children;
		for (var i = 0; i < el.length; i++) {
			var e = el[i];
			var a = find_child(e, "rect").attributes;
			var ex = parseFloat(a.x.value);
			var ew = parseFloat(a.width.value);
			var upstack;
			// Is it an ancestor
			if (0 == 0) {
				upstack = parseFloat(a.y.value) > ymin;
			} else {
				upstack = parseFloat(a.y.value) < ymin;
			}
			if (upstack) {
				// Direct ancestor
				if (ex <= xmin && (ex+ew+fudge) >= xmax) {
					e.classList.add("parent");
					zoom_parent(e);
					update_text(e);
				}
				// not in current path
				else
					e.classList.add("hide");
			}
			// Children maybe
			else {
				// no common path
				if (ex < xmin || ex + fudge >= xmax) {
					e.classList.add("hide");
				}
				else {
					zoom_child(e, xmin, ratio);
					update_text(e);
				}
			}
		}
		search();
	}
	function unzoom() {
		unzoombtn.classList.add("hide");
		var el = document.getElementById("frames").children;
		for(var i = 0; i < el.length; i++) {
			el[i].classList.remove("parent");
			el[i].classList.remove("hide");
			zoom_reset(el[i]);
			update_text(el[i]);
		}
		search();
	}

	// search
	function toggle_ignorecase() {
		ignorecase = !ignorecase;
		if (ignorecase) {
			ignorecaseBtn.classList.add("show");
		} else {
			ignorecaseBtn.classList.remove("show");
		}
		reset_search();
		search();
	}
	function reset_search() {
		var el = document.querySelectorAll("#frames rect");
		for (var i = 0; i < el.length; i++) {
			orig_load(el[i], "fill")
		}
	}
	function search_prompt() {
		if (!searching) {
			var term = prompt("Enter a search term (regexp " +
			    "allowed, eg: ^ext4_)"
			    + (ignorecase ? ", ignoring case" : "")
			    + "\nPress Ctrl-i to toggle case sensitivity", "");
			if (term != null) {
				currentSearchTerm = term;
				search();
			}
		} else {
			reset_search();
			searching = 0;
			currentSearchTerm = null;
			searchbtn.classList.remove("show");
			searchbtn.firstChild.nodeValue = "Search"
			matchedtxt.classList.add("hide");
			matchedtxt.firstChild.nodeValue = ""
		}
	}
	function search(term) {
		if (currentSearchTerm === null) return;
		var term = currentSearchTerm;

		var re = new RegExp(term, ignorecase ? 'i' : '');
		var el = document.getElementById("frames").children;
		var matches = new Object();
		var maxwidth = 0;
		for (var i = 0; i < el.length; i++) {
			var e = el[i];
			var func = g_to_func(e);
			var rect = find_child(e, "rect");
			if (func == null || rect == null)
				continue;

			// Save max width. Only works as we have a root frame
			var w = parseFloat(rect.attributes.width.value);
			if (w > maxwidth)
				maxwidth = w;

			if (func.match(re)) {
				// highlight
				var x = parseFloat(rect.attributes.x.value);
				orig_save(rect, "fill");
				rect.attributes.fill.value = "rgb(230,0,230)";

				// remember matches
				if (matches[x] == undefined) {
					matches[x] = w;
				} else {
					if (w > matches[x]) {
						// overwrite with parent
						matches[x] = w;
					}
				}
				searching = 1;
			}
		}
		if (!searching)
			return;

		searchbtn.classList.add("show");
		searchbtn.firstChild.nodeValue = "Reset Search";

		// calculate percent matched, excluding vertical overlap
		var count = 0;
		var lastx = -1;
		var lastw = 0;
		var keys = Array();
		for (k in matches) {
			if (matches.hasOwnProperty(k))
				keys.push(k);
		}
		// sort the matched frames by their x location
		// ascending, then width descending
		keys.sort(function(a, b){
			return a - b;
		});
		// Step through frames saving only the biggest bottom-up frames
		// thanks to the sort order. This relies on the tree property
		// where children are always smaller than their parents.
		var fudge = 0.0001;	// JavaScript floating point
		for (var k in keys) {
			var x = parseFloat(keys[k]);
			var w = matches[keys[k]];
			if (x >= lastx + lastw - fudge) {
				count += w;
				lastx = x;
				lastw = w;
			}
		}
		// display matched percent
		matchedtxt.classList.remove("hide");
		var pct = 100 * count / maxwidth;
		if (pct != 100) pct = pct.toFixed(1)
		matchedtxt.firstChild.nodeValue = "Matched: " + pct + "%";
	}
]]>
</script>
<rect x="0.0" y="0" width="1200.0" height="966.0" fill="url(#background)"  />
<text id="title" x="600.00" y="24" >Flame Graph</text>
<text id="details" x="10.00" y="949" > </text>
<text id="unzoom" x="10.00" y="24" class="hide">Reset Zoom</text>
<text id="search" x="1090.00" y="24" >Search</text>
<text id="ignorecase" x="1174.00" y="24" >ic</text>
<text id="matched" x="1090.00" y="949" > </text>
<g id="frames">
<g >
<title>runtime.dodeltimer0 (61 samples, 0.02%)</title><rect x="1081.9" y="581" width="0.3" height="15.0" fill="rgb(247,225,10)" rx="2" ry="2" />
<text  x="1084.94" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyOutBytes (954 samples, 0.29%)</title><rect x="73.4" y="741" width="3.4" height="15.0" fill="rgb(238,56,6)" rx="2" ry="2" />
<text  x="76.36" y="751.5" ></text>
</g>
<g >
<title>runtime.gosched_m (62 samples, 0.02%)</title><rect x="54.4" y="693" width="0.3" height="15.0" fill="rgb(215,215,10)" rx="2" ry="2" />
<text  x="57.44" y="703.5" ></text>
</g>
<g >
<title>runtime.newobject (93 samples, 0.03%)</title><rect x="1076.4" y="709" width="0.4" height="15.0" fill="rgb(235,203,44)" rx="2" ry="2" />
<text  x="1079.43" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Stat (1,727 samples, 0.53%)</title><rect x="183.6" y="773" width="6.3" height="15.0" fill="rgb(210,16,8)" rx="2" ry="2" />
<text  x="186.63" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*packetBufferQueue).dequeue (94 samples, 0.03%)</title><rect x="998.7" y="837" width="0.3" height="15.0" fill="rgb(254,210,18)" rx="2" ry="2" />
<text  x="1001.67" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (187 samples, 0.06%)</title><rect x="1082.8" y="533" width="0.7" height="15.0" fill="rgb(250,126,49)" rx="2" ry="2" />
<text  x="1085.79" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).Capabilities (48 samples, 0.01%)</title><rect x="723.2" y="709" width="0.2" height="15.0" fill="rgb(235,134,35)" rx="2" ry="2" />
<text  x="726.18" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleTimeWaitSegments (325 samples, 0.10%)</title><rect x="1067.3" y="837" width="1.2" height="15.0" fill="rgb(228,44,17)" rx="2" ry="2" />
<text  x="1070.29" y="847.5" ></text>
</g>
<g >
<title>type..hash.gvisor.dev/gvisor/pkg/tcpip/stack.TransportEndpointID (74 samples, 0.02%)</title><rect x="773.0" y="645" width="0.3" height="15.0" fill="rgb(226,20,19)" rx="2" ry="2" />
<text  x="776.01" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="1028.6" y="357" width="0.2" height="15.0" fill="rgb(241,5,20)" rx="2" ry="2" />
<text  x="1031.58" y="367.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (47 samples, 0.01%)</title><rect x="149.2" y="469" width="0.2" height="15.0" fill="rgb(224,37,51)" rx="2" ry="2" />
<text  x="152.18" y="479.5" ></text>
</g>
<g >
<title>exe (326,293 samples, 100.00%)</title><rect x="10.0" y="901" width="1180.0" height="15.0" fill="rgb(213,163,11)" rx="2" ry="2" />
<text  x="13.00" y="911.5" >exe</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (41 samples, 0.01%)</title><rect x="1073.8" y="533" width="0.2" height="15.0" fill="rgb(242,220,30)" rx="2" ry="2" />
<text  x="1076.81" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/usermem.AddrRangeSeq.DropFirst64 (31 samples, 0.01%)</title><rect x="206.5" y="597" width="0.1" height="15.0" fill="rgb(210,41,27)" rx="2" ry="2" />
<text  x="209.51" y="607.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (66 samples, 0.02%)</title><rect x="930.1" y="773" width="0.2" height="15.0" fill="rgb(213,102,6)" rx="2" ry="2" />
<text  x="933.09" y="783.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (125 samples, 0.04%)</title><rect x="1003.6" y="709" width="0.4" height="15.0" fill="rgb(250,80,25)" rx="2" ry="2" />
<text  x="1006.56" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (81 samples, 0.02%)</title><rect x="1005.8" y="741" width="0.3" height="15.0" fill="rgb(215,142,23)" rx="2" ry="2" />
<text  x="1008.79" y="751.5" ></text>
</g>
<g >
<title>runtime.mcall (414 samples, 0.13%)</title><rect x="17.2" y="853" width="1.5" height="15.0" fill="rgb(211,156,48)" rx="2" ry="2" />
<text  x="20.18" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (63 samples, 0.02%)</title><rect x="1189.8" y="629" width="0.2" height="15.0" fill="rgb(226,204,49)" rx="2" ry="2" />
<text  x="1192.76" y="639.5" ></text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (152 samples, 0.05%)</title><rect x="847.5" y="725" width="0.6" height="15.0" fill="rgb(215,106,52)" rx="2" ry="2" />
<text  x="850.54" y="735.5" ></text>
</g>
<g >
<title>runtime.assertI2I2 (267 samples, 0.08%)</title><rect x="791.5" y="709" width="1.0" height="15.0" fill="rgb(229,140,32)" rx="2" ry="2" />
<text  x="794.51" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*SocketOperations).Accept (3,211 samples, 0.98%)</title><rect x="44.9" y="741" width="11.6" height="15.0" fill="rgb(232,82,32)" rx="2" ry="2" />
<text  x="47.87" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).StartTransportEndpointCleanup (1,191 samples, 0.37%)</title><rect x="1101.0" y="821" width="4.3" height="15.0" fill="rgb(223,131,14)" rx="2" ry="2" />
<text  x="1104.02" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).accountTaskGoroutineLeave (66 samples, 0.02%)</title><rect x="85.3" y="709" width="0.3" height="15.0" fill="rgb(244,33,22)" rx="2" ry="2" />
<text  x="88.33" y="719.5" ></text>
</g>
<g >
<title>runtime.unlock2 (81 samples, 0.02%)</title><rect x="158.2" y="645" width="0.3" height="15.0" fill="rgb(241,131,1)" rx="2" ry="2" />
<text  x="161.16" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).wait (107 samples, 0.03%)</title><rect x="638.9" y="821" width="0.4" height="15.0" fill="rgb(238,14,48)" rx="2" ry="2" />
<text  x="641.90" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (83 samples, 0.03%)</title><rect x="737.1" y="629" width="0.3" height="15.0" fill="rgb(240,134,43)" rx="2" ry="2" />
<text  x="740.11" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).Destroy (181 samples, 0.06%)</title><rect x="1050.8" y="757" width="0.7" height="15.0" fill="rgb(228,163,21)" rx="2" ry="2" />
<text  x="1053.80" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,903 samples, 0.58%)</title><rect x="194.5" y="453" width="6.9" height="15.0" fill="rgb(240,136,2)" rx="2" ry="2" />
<text  x="197.52" y="463.5" ></text>
</g>
<g >
<title>runtime.schedule (507 samples, 0.16%)</title><rect x="1107.5" y="805" width="1.8" height="15.0" fill="rgb(231,119,10)" rx="2" ry="2" />
<text  x="1110.51" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (647 samples, 0.20%)</title><rect x="693.3" y="629" width="2.3" height="15.0" fill="rgb(222,178,17)" rx="2" ry="2" />
<text  x="696.27" y="639.5" ></text>
</g>
<g >
<title>runtime.mallocgc (74 samples, 0.02%)</title><rect x="762.6" y="581" width="0.3" height="15.0" fill="rgb(205,72,27)" rx="2" ry="2" />
<text  x="765.62" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (519 samples, 0.16%)</title><rect x="59.2" y="757" width="1.9" height="15.0" fill="rgb(242,7,4)" rx="2" ry="2" />
<text  x="62.18" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket.ConvertAddress (53 samples, 0.02%)</title><rect x="47.1" y="725" width="0.2" height="15.0" fill="rgb(206,64,6)" rx="2" ry="2" />
<text  x="50.13" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).runGoroutine (1,825 samples, 0.56%)</title><rect x="697.2" y="869" width="6.6" height="15.0" fill="rgb(207,208,13)" rx="2" ry="2" />
<text  x="700.17" y="879.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.writeAddress (301 samples, 0.09%)</title><rect x="56.5" y="741" width="1.1" height="15.0" fill="rgb(221,93,29)" rx="2" ry="2" />
<text  x="59.49" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).get (111 samples, 0.03%)</title><rect x="78.7" y="725" width="0.4" height="15.0" fill="rgb(215,142,27)" rx="2" ry="2" />
<text  x="81.73" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (529 samples, 0.16%)</title><rect x="196.0" y="325" width="1.9" height="15.0" fill="rgb(217,222,4)" rx="2" ry="2" />
<text  x="199.01" y="335.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).existingPMAsLocked (86 samples, 0.03%)</title><rect x="62.6" y="693" width="0.3" height="15.0" fill="rgb(223,117,22)" rx="2" ry="2" />
<text  x="65.58" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*SocketOperations).Write (3,164 samples, 0.97%)</title><rect x="205.1" y="725" width="11.5" height="15.0" fill="rgb(227,94,43)" rx="2" ry="2" />
<text  x="208.14" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="18.1" y="645" width="0.1" height="15.0" fill="rgb(205,180,43)" rx="2" ry="2" />
<text  x="21.06" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendRaw (1,354 samples, 0.41%)</title><rect x="1072.0" y="757" width="4.9" height="15.0" fill="rgb(250,60,52)" rx="2" ry="2" />
<text  x="1075.04" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (445 samples, 0.14%)</title><rect x="988.1" y="133" width="1.6" height="15.0" fill="rgb(234,49,48)" rx="2" ry="2" />
<text  x="991.07" y="143.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (78 samples, 0.02%)</title><rect x="18.4" y="693" width="0.3" height="15.0" fill="rgb(242,224,50)" rx="2" ry="2" />
<text  x="21.38" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,935 samples, 1.82%)</title><rect x="425.2" y="661" width="21.5" height="15.0" fill="rgb(244,94,34)" rx="2" ry="2" />
<text  x="428.19" y="671.5" >[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).DeliverNetworkPacket (56 samples, 0.02%)</title><rect x="796.2" y="773" width="0.2" height="15.0" fill="rgb(213,190,49)" rx="2" ry="2" />
<text  x="799.24" y="783.5" ></text>
</g>
<g >
<title>runtime.mallocgc (127 samples, 0.04%)</title><rect x="730.6" y="677" width="0.5" height="15.0" fill="rgb(233,172,15)" rx="2" ry="2" />
<text  x="733.63" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (161 samples, 0.05%)</title><rect x="451.7" y="645" width="0.5" height="15.0" fill="rgb(212,80,44)" rx="2" ry="2" />
<text  x="454.65" y="655.5" ></text>
</g>
<g >
<title>runtime.runqsteal (2,421 samples, 0.74%)</title><rect x="118.7" y="629" width="8.8" height="15.0" fill="rgb(224,42,22)" rx="2" ry="2" />
<text  x="121.73" y="639.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast64 (158 samples, 0.05%)</title><rect x="774.4" y="677" width="0.6" height="15.0" fill="rgb(222,74,35)" rx="2" ry="2" />
<text  x="777.38" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*TcpipTimer).Stop (350 samples, 0.11%)</title><rect x="1050.6" y="773" width="1.3" height="15.0" fill="rgb(251,43,0)" rx="2" ry="2" />
<text  x="1053.64" y="783.5" ></text>
</g>
<g >
<title>runtime.mallocgc (52 samples, 0.02%)</title><rect x="1081.6" y="613" width="0.2" height="15.0" fill="rgb(226,120,7)" rx="2" ry="2" />
<text  x="1084.61" y="623.5" ></text>
</g>
<g >
<title>syscall.wait4 (46 samples, 0.01%)</title><rect x="451.1" y="789" width="0.1" height="15.0" fill="rgb(234,63,34)" rx="2" ry="2" />
<text  x="454.07" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/abi/linux.CopyEpollEventSliceOut (1,039 samples, 0.32%)</title><rect x="73.2" y="757" width="3.7" height="15.0" fill="rgb(215,156,43)" rx="2" ry="2" />
<text  x="76.19" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendSegmentFromView (776 samples, 0.24%)</title><rect x="180.2" y="629" width="2.8" height="15.0" fill="rgb(241,159,17)" rx="2" ry="2" />
<text  x="183.19" y="639.5" ></text>
</g>
<g >
<title>reflect.Value.Field (173 samples, 0.05%)</title><rect x="925.0" y="773" width="0.6" height="15.0" fill="rgb(239,43,16)" rx="2" ry="2" />
<text  x="927.98" y="783.5" ></text>
</g>
<g >
<title>runtime.newproc1 (57 samples, 0.02%)</title><rect x="904.0" y="645" width="0.2" height="15.0" fill="rgb(233,68,16)" rx="2" ry="2" />
<text  x="907.01" y="655.5" ></text>
</g>
<g >
<title>runtime.mcall (32 samples, 0.01%)</title><rect x="52.6" y="693" width="0.1" height="15.0" fill="rgb(232,167,54)" rx="2" ry="2" />
<text  x="55.61" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.PseudoHeaderChecksum (52 samples, 0.02%)</title><rect x="1030.5" y="725" width="0.2" height="15.0" fill="rgb(232,228,40)" rx="2" ry="2" />
<text  x="1033.52" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (73 samples, 0.02%)</title><rect x="23.0" y="661" width="0.3" height="15.0" fill="rgb(250,199,30)" rx="2" ry="2" />
<text  x="26.01" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/abi/linux.(*EpollEvent).CopyIn (141 samples, 0.04%)</title><rect x="69.6" y="757" width="0.5" height="15.0" fill="rgb(210,199,36)" rx="2" ry="2" />
<text  x="72.62" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (253 samples, 0.08%)</title><rect x="1127.6" y="725" width="1.0" height="15.0" fill="rgb(206,111,49)" rx="2" ry="2" />
<text  x="1130.65" y="735.5" ></text>
</g>
<g >
<title>runtime.newproc.func1 (61 samples, 0.02%)</title><rect x="904.0" y="661" width="0.2" height="15.0" fill="rgb(232,86,23)" rx="2" ry="2" />
<text  x="906.99" y="671.5" ></text>
</g>
<g >
<title>runtime.mallocgc (87 samples, 0.03%)</title><rect x="183.1" y="677" width="0.3" height="15.0" fill="rgb(217,167,27)" rx="2" ry="2" />
<text  x="186.11" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.IPv4.CalculateChecksum (227 samples, 0.07%)</title><rect x="731.2" y="709" width="0.9" height="15.0" fill="rgb(224,22,14)" rx="2" ry="2" />
<text  x="734.24" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (177 samples, 0.05%)</title><rect x="1119.1" y="725" width="0.6" height="15.0" fill="rgb(244,190,26)" rx="2" ry="2" />
<text  x="1122.09" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (61 samples, 0.02%)</title><rect x="1027.8" y="597" width="0.2" height="15.0" fill="rgb(237,145,36)" rx="2" ry="2" />
<text  x="1030.81" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (556 samples, 0.17%)</title><rect x="916.7" y="533" width="2.0" height="15.0" fill="rgb(233,66,26)" rx="2" ry="2" />
<text  x="919.72" y="543.5" ></text>
</g>
<g >
<title>[unknown] (47 samples, 0.01%)</title><rect x="16.8" y="789" width="0.1" height="15.0" fill="rgb(212,135,46)" rx="2" ry="2" />
<text  x="19.75" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).run (183,537 samples, 56.25%)</title><rect x="33.1" y="869" width="663.7" height="15.0" fill="rgb(211,113,52)" rx="2" ry="2" />
<text  x="36.07" y="879.5" >gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).run</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*PTrace).CooperativelySchedulesAddressSpace (35 samples, 0.01%)</title><rect x="84.6" y="677" width="0.1" height="15.0" fill="rgb(236,143,9)" rx="2" ry="2" />
<text  x="87.57" y="687.5" ></text>
</g>
<g >
<title>runtime.mallocgc (32 samples, 0.01%)</title><rect x="1079.9" y="629" width="0.1" height="15.0" fill="rgb(229,62,14)" rx="2" ry="2" />
<text  x="1082.91" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*inodeOperations).Getlink (31 samples, 0.01%)</title><rect x="189.2" y="645" width="0.2" height="15.0" fill="rgb(221,66,13)" rx="2" ry="2" />
<text  x="192.24" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (312 samples, 0.10%)</title><rect x="59.8" y="725" width="1.1" height="15.0" fill="rgb(220,68,23)" rx="2" ry="2" />
<text  x="62.78" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindRoute (1,517 samples, 0.46%)</title><rect x="1032.5" y="789" width="5.5" height="15.0" fill="rgb(206,29,11)" rx="2" ry="2" />
<text  x="1035.55" y="799.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (287 samples, 0.09%)</title><rect x="849.3" y="821" width="1.0" height="15.0" fill="rgb(236,206,43)" rx="2" ry="2" />
<text  x="852.27" y="831.5" ></text>
</g>
<g >
<title>runtime.mallocgc (412 samples, 0.13%)</title><rect x="710.3" y="789" width="1.5" height="15.0" fill="rgb(231,207,3)" rx="2" ry="2" />
<text  x="713.29" y="799.5" ></text>
</g>
<g >
<title>runtime.systemstack (5,168 samples, 1.58%)</title><rect x="823.3" y="805" width="18.7" height="15.0" fill="rgb(250,196,20)" rx="2" ry="2" />
<text  x="826.33" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="18.5" y="581" width="0.2" height="15.0" fill="rgb(226,147,16)" rx="2" ry="2" />
<text  x="21.54" y="591.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc.func1 (57 samples, 0.02%)</title><rect x="1020.2" y="789" width="0.2" height="15.0" fill="rgb(245,80,42)" rx="2" ry="2" />
<text  x="1023.16" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (76 samples, 0.02%)</title><rect x="1061.2" y="725" width="0.3" height="15.0" fill="rgb(229,138,9)" rx="2" ry="2" />
<text  x="1064.22" y="735.5" ></text>
</g>
<g >
<title>runtime.assertI2I2 (66 samples, 0.02%)</title><rect x="179.0" y="741" width="0.2" height="15.0" fill="rgb(254,50,29)" rx="2" ry="2" />
<text  x="181.97" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/syserr.(*Error).ToError (41 samples, 0.01%)</title><rect x="178.8" y="741" width="0.2" height="15.0" fill="rgb(208,17,26)" rx="2" ry="2" />
<text  x="181.82" y="751.5" ></text>
</g>
<g >
<title>indexbytebody (82 samples, 0.03%)</title><rect x="1112.8" y="805" width="0.3" height="15.0" fill="rgb(245,17,29)" rx="2" ry="2" />
<text  x="1115.83" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).FeatureSet (43 samples, 0.01%)</title><rect x="222.7" y="821" width="0.1" height="15.0" fill="rgb(218,197,9)" rx="2" ry="2" />
<text  x="225.68" y="831.5" ></text>
</g>
<g >
<title>runtime.newobject (59 samples, 0.02%)</title><rect x="1031.2" y="741" width="0.2" height="15.0" fill="rgb(254,86,27)" rx="2" ry="2" />
<text  x="1034.16" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).setFPRegs (48 samples, 0.01%)</title><rect x="638.7" y="821" width="0.2" height="15.0" fill="rgb(253,173,23)" rx="2" ry="2" />
<text  x="641.69" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/buffer.(*VectorisedView).TrimFront (41 samples, 0.01%)</title><rect x="709.6" y="821" width="0.2" height="15.0" fill="rgb(242,107,4)" rx="2" ry="2" />
<text  x="712.64" y="831.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_faststr (61 samples, 0.02%)</title><rect x="187.3" y="677" width="0.2" height="15.0" fill="rgb(237,64,27)" rx="2" ry="2" />
<text  x="190.25" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (123 samples, 0.04%)</title><rect x="15.4" y="581" width="0.5" height="15.0" fill="rgb(238,140,42)" rx="2" ry="2" />
<text  x="18.42" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header/parse.IPv4 (134 samples, 0.04%)</title><rect x="780.5" y="725" width="0.5" height="15.0" fill="rgb(246,186,3)" rx="2" ry="2" />
<text  x="783.54" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).UnstableAttr (104 samples, 0.03%)</title><rect x="191.1" y="709" width="0.4" height="15.0" fill="rgb(248,44,29)" rx="2" ry="2" />
<text  x="194.13" y="719.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (35 samples, 0.01%)</title><rect x="1094.3" y="725" width="0.1" height="15.0" fill="rgb(207,43,43)" rx="2" ry="2" />
<text  x="1097.28" y="735.5" ></text>
</g>
<g >
<title>runtime.casgstatus (34 samples, 0.01%)</title><rect x="1003.6" y="677" width="0.1" height="15.0" fill="rgb(231,5,14)" rx="2" ry="2" />
<text  x="1006.61" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendSegmentFromView (1,400 samples, 0.43%)</title><rect x="1072.0" y="773" width="5.0" height="15.0" fill="rgb(206,41,9)" rx="2" ry="2" />
<text  x="1074.97" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (33 samples, 0.01%)</title><rect x="722.3" y="741" width="0.2" height="15.0" fill="rgb(239,175,0)" rx="2" ry="2" />
<text  x="725.33" y="751.5" ></text>
</g>
<g >
<title>runtime.casgstatus (65 samples, 0.02%)</title><rect x="823.0" y="789" width="0.2" height="15.0" fill="rgb(218,5,54)" rx="2" ry="2" />
<text  x="825.95" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).enqueueSegment (273 samples, 0.08%)</title><rect x="759.4" y="629" width="1.0" height="15.0" fill="rgb(219,72,27)" rx="2" ry="2" />
<text  x="762.40" y="639.5" ></text>
</g>
<g >
<title>runtime.stopm (68 samples, 0.02%)</title><rect x="1189.7" y="805" width="0.3" height="15.0" fill="rgb(237,208,8)" rx="2" ry="2" />
<text  x="1192.75" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (64 samples, 0.02%)</title><rect x="1189.8" y="645" width="0.2" height="15.0" fill="rgb(247,152,31)" rx="2" ry="2" />
<text  x="1192.76" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (51 samples, 0.02%)</title><rect x="1061.3" y="709" width="0.2" height="15.0" fill="rgb(244,109,24)" rx="2" ry="2" />
<text  x="1064.31" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).executeSyscall (55 samples, 0.02%)</title><rect x="21.6" y="853" width="0.2" height="15.0" fill="rgb(219,99,43)" rx="2" ry="2" />
<text  x="24.56" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (42 samples, 0.01%)</title><rect x="1079.1" y="645" width="0.1" height="15.0" fill="rgb(236,162,31)" rx="2" ry="2" />
<text  x="1082.06" y="655.5" ></text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (48 samples, 0.01%)</title><rect x="50.7" y="581" width="0.2" height="15.0" fill="rgb(213,78,48)" rx="2" ry="2" />
<text  x="53.69" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (28 samples, 0.01%)</title><rect x="760.1" y="581" width="0.1" height="15.0" fill="rgb(245,158,21)" rx="2" ry="2" />
<text  x="763.12" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (824 samples, 0.25%)</title><rect x="153.3" y="469" width="3.0" height="15.0" fill="rgb(254,173,11)" rx="2" ry="2" />
<text  x="156.31" y="479.5" ></text>
</g>
<g >
<title>runtime.futex (93 samples, 0.03%)</title><rect x="701.7" y="741" width="0.3" height="15.0" fill="rgb(211,53,37)" rx="2" ry="2" />
<text  x="704.66" y="751.5" ></text>
</g>
<g >
<title>runtime.findfunc (59 samples, 0.02%)</title><rect x="1103.8" y="709" width="0.3" height="15.0" fill="rgb(223,104,31)" rx="2" ry="2" />
<text  x="1106.84" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,598 samples, 1.41%)</title><rect x="824.6" y="709" width="16.6" height="15.0" fill="rgb(222,42,11)" rx="2" ry="2" />
<text  x="827.62" y="719.5" ></text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (40 samples, 0.01%)</title><rect x="168.8" y="725" width="0.1" height="15.0" fill="rgb(230,161,48)" rx="2" ry="2" />
<text  x="171.78" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).Shutdown (1,082 samples, 0.33%)</title><rect x="179.6" y="757" width="3.9" height="15.0" fill="rgb(214,48,37)" rx="2" ry="2" />
<text  x="182.55" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).FloatingPointData (51 samples, 0.02%)</title><rect x="227.4" y="805" width="0.2" height="15.0" fill="rgb(247,228,48)" rx="2" ry="2" />
<text  x="230.37" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*MountNamespace).FindInode (1,276 samples, 0.39%)</title><rect x="185.0" y="741" width="4.6" height="15.0" fill="rgb(239,177,28)" rx="2" ry="2" />
<text  x="188.00" y="751.5" ></text>
</g>
<g >
<title>runtime.schedule (28 samples, 0.01%)</title><rect x="1021.6" y="789" width="0.1" height="15.0" fill="rgb(208,94,0)" rx="2" ry="2" />
<text  x="1024.65" y="799.5" ></text>
</g>
<g >
<title>runtime.mcall (44 samples, 0.01%)</title><rect x="696.8" y="837" width="0.2" height="15.0" fill="rgb(232,213,28)" rx="2" ry="2" />
<text  x="699.81" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (80 samples, 0.02%)</title><rect x="993.3" y="261" width="0.3" height="15.0" fill="rgb(220,169,13)" rx="2" ry="2" />
<text  x="996.29" y="271.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*EventPoll).ReadEvents (61 samples, 0.02%)</title><rect x="77.3" y="757" width="0.2" height="15.0" fill="rgb(233,155,12)" rx="2" ry="2" />
<text  x="80.27" y="767.5" ></text>
</g>
<g >
<title>runtime.makeslice (118 samples, 0.04%)</title><rect x="1011.5" y="757" width="0.4" height="15.0" fill="rgb(210,85,30)" rx="2" ry="2" />
<text  x="1014.49" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*pmaSet).Find (41 samples, 0.01%)</title><rect x="76.4" y="693" width="0.2" height="15.0" fill="rgb(251,129,19)" rx="2" ry="2" />
<text  x="79.42" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*listenContext).startHandshake (5,835 samples, 1.79%)</title><rect x="1022.2" y="821" width="21.1" height="15.0" fill="rgb(211,152,13)" rx="2" ry="2" />
<text  x="1025.25" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*handshake).synRcvdState (3,033 samples, 0.93%)</title><rect x="1008.2" y="805" width="11.0" height="15.0" fill="rgb(248,221,9)" rx="2" ry="2" />
<text  x="1011.22" y="815.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (62 samples, 0.02%)</title><rect x="1012.8" y="725" width="0.2" height="15.0" fill="rgb(224,102,12)" rx="2" ry="2" />
<text  x="1015.82" y="735.5" ></text>
</g>
<g >
<title>runtime.futex (123 samples, 0.04%)</title><rect x="1028.3" y="453" width="0.5" height="15.0" fill="rgb(230,187,24)" rx="2" ry="2" />
<text  x="1031.33" y="463.5" ></text>
</g>
<g >
<title>runtime.systemstack (57 samples, 0.02%)</title><rect x="1020.2" y="805" width="0.2" height="15.0" fill="rgb(212,144,39)" rx="2" ry="2" />
<text  x="1023.16" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*pollEntry).Callback (974 samples, 0.30%)</title><rect x="1000.6" y="821" width="3.5" height="15.0" fill="rgb(222,188,27)" rx="2" ry="2" />
<text  x="1003.62" y="831.5" ></text>
</g>
<g >
<title>runtime.mallocgc (92 samples, 0.03%)</title><rect x="729.0" y="661" width="0.3" height="15.0" fill="rgb(209,111,54)" rx="2" ry="2" />
<text  x="731.97" y="671.5" ></text>
</g>
<g >
<title>runtime.goexit0 (4,734 samples, 1.45%)</title><rect x="1111.9" y="853" width="17.1" height="15.0" fill="rgb(225,71,7)" rx="2" ry="2" />
<text  x="1114.85" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.buildTCPHdr (85 samples, 0.03%)</title><rect x="1075.9" y="709" width="0.3" height="15.0" fill="rgb(205,46,12)" rx="2" ry="2" />
<text  x="1078.86" y="719.5" ></text>
</g>
<g >
<title>runtime.futex (131 samples, 0.04%)</title><rect x="13.5" y="725" width="0.5" height="15.0" fill="rgb(214,194,5)" rx="2" ry="2" />
<text  x="16.50" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).Fetch (385 samples, 0.12%)</title><rect x="1006.5" y="837" width="1.4" height="15.0" fill="rgb(234,13,6)" rx="2" ry="2" />
<text  x="1009.47" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (455 samples, 0.14%)</title><rect x="61.5" y="709" width="1.6" height="15.0" fill="rgb(226,137,5)" rx="2" ry="2" />
<text  x="64.47" y="719.5" ></text>
</g>
<g >
<title>runtime.wakeNetPoller (195 samples, 0.06%)</title><rect x="1049.0" y="661" width="0.7" height="15.0" fill="rgb(245,18,13)" rx="2" ry="2" />
<text  x="1052.04" y="671.5" ></text>
</g>
<g >
<title>sync.(*poolChain).popTail (38 samples, 0.01%)</title><rect x="45.9" y="661" width="0.1" height="15.0" fill="rgb(239,56,34)" rx="2" ry="2" />
<text  x="48.88" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.unrolledCalculateChecksum (44 samples, 0.01%)</title><rect x="1030.5" y="709" width="0.2" height="15.0" fill="rgb(219,29,0)" rx="2" ry="2" />
<text  x="1033.53" y="719.5" ></text>
</g>
<g >
<title>runtime.notewakeup (90 samples, 0.03%)</title><rect x="19.7" y="613" width="0.3" height="15.0" fill="rgb(213,181,22)" rx="2" ry="2" />
<text  x="22.66" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendRaw (746 samples, 0.23%)</title><rect x="180.2" y="613" width="2.7" height="15.0" fill="rgb(215,13,26)" rx="2" ry="2" />
<text  x="183.22" y="623.5" ></text>
</g>
<g >
<title>tcp_packet (95 samples, 0.03%)</title><rect x="991.7" y="165" width="0.4" height="15.0" fill="rgb(252,110,22)" rx="2" ry="2" />
<text  x="994.72" y="175.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,687 samples, 0.52%)</title><rect x="150.2" y="565" width="6.1" height="15.0" fill="rgb(205,183,3)" rx="2" ry="2" />
<text  x="153.19" y="575.5" ></text>
</g>
<g >
<title>runtime.park_m (219 samples, 0.07%)</title><rect x="1045.8" y="805" width="0.8" height="15.0" fill="rgb(232,14,53)" rx="2" ry="2" />
<text  x="1048.79" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (81 samples, 0.02%)</title><rect x="84.9" y="677" width="0.3" height="15.0" fill="rgb(243,149,22)" rx="2" ry="2" />
<text  x="87.95" y="687.5" ></text>
</g>
<g >
<title>ext4_mark_inode_dirty (207 samples, 0.06%)</title><rect x="200.1" y="245" width="0.8" height="15.0" fill="rgb(245,18,6)" rx="2" ry="2" />
<text  x="203.13" y="255.5" ></text>
</g>
<g >
<title>runtime.mstart (878 samples, 0.27%)</title><rect x="12.7" y="805" width="3.2" height="15.0" fill="rgb(220,192,7)" rx="2" ry="2" />
<text  x="15.72" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (61 samples, 0.02%)</title><rect x="726.9" y="661" width="0.2" height="15.0" fill="rgb(230,179,5)" rx="2" ry="2" />
<text  x="729.85" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (622 samples, 0.19%)</title><rect x="125.1" y="469" width="2.3" height="15.0" fill="rgb(234,66,34)" rx="2" ry="2" />
<text  x="128.12" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (32 samples, 0.01%)</title><rect x="1079.1" y="613" width="0.1" height="15.0" fill="rgb(218,95,5)" rx="2" ry="2" />
<text  x="1082.09" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/auth.CredentialsFromContext (42 samples, 0.01%)</title><rect x="52.3" y="693" width="0.1" height="15.0" fill="rgb(217,27,41)" rx="2" ry="2" />
<text  x="55.28" y="703.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (139 samples, 0.04%)</title><rect x="848.7" y="789" width="0.5" height="15.0" fill="rgb(215,9,37)" rx="2" ry="2" />
<text  x="851.67" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendTCP (1,264 samples, 0.39%)</title><rect x="1072.2" y="741" width="4.6" height="15.0" fill="rgb(244,110,3)" rx="2" ry="2" />
<text  x="1075.24" y="751.5" ></text>
</g>
<g >
<title>runtime.mapdelete (43 samples, 0.01%)</title><rect x="1101.7" y="773" width="0.1" height="15.0" fill="rgb(242,126,54)" rx="2" ry="2" />
<text  x="1104.66" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (61 samples, 0.02%)</title><rect x="726.9" y="677" width="0.2" height="15.0" fill="rgb(205,147,48)" rx="2" ry="2" />
<text  x="729.85" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).AfterFunc (1,473 samples, 0.45%)</title><rect x="1078.9" y="709" width="5.3" height="15.0" fill="rgb(234,198,16)" rx="2" ry="2" />
<text  x="1081.87" y="719.5" ></text>
</g>
<g >
<title>runtime.newproc.func1 (99 samples, 0.03%)</title><rect x="1048.1" y="693" width="0.3" height="15.0" fill="rgb(222,152,52)" rx="2" ry="2" />
<text  x="1051.09" y="703.5" ></text>
</g>
<g >
<title>runtime.newobject (1,088 samples, 0.33%)</title><rect x="764.3" y="613" width="3.9" height="15.0" fill="rgb(215,115,43)" rx="2" ry="2" />
<text  x="767.31" y="623.5" ></text>
</g>
<g >
<title>fdb_find_rcu (92 samples, 0.03%)</title><rect x="994.8" y="437" width="0.3" height="15.0" fill="rgb(214,221,2)" rx="2" ry="2" />
<text  x="997.81" y="447.5" ></text>
</g>
<g >
<title>runtime.closechan (116 samples, 0.04%)</title><rect x="1085.1" y="677" width="0.4" height="15.0" fill="rgb(227,90,26)" rx="2" ry="2" />
<text  x="1088.07" y="687.5" ></text>
</g>
<g >
<title>runtime.newproc1 (128 samples, 0.04%)</title><rect x="1043.7" y="789" width="0.5" height="15.0" fill="rgb(242,190,53)" rx="2" ry="2" />
<text  x="1046.74" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (13,182 samples, 4.04%)</title><rect x="949.0" y="581" width="47.7" height="15.0" fill="rgb(233,193,15)" rx="2" ry="2" />
<text  x="952.03" y="591.5" >[[ke..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (226 samples, 0.07%)</title><rect x="700.8" y="709" width="0.8" height="15.0" fill="rgb(226,108,13)" rx="2" ry="2" />
<text  x="703.77" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (207 samples, 0.06%)</title><rect x="700.8" y="693" width="0.8" height="15.0" fill="rgb(252,179,38)" rx="2" ry="2" />
<text  x="703.84" y="703.5" ></text>
</g>
<g >
<title>runtime.newobject (37 samples, 0.01%)</title><rect x="46.2" y="709" width="0.1" height="15.0" fill="rgb(226,206,11)" rx="2" ry="2" />
<text  x="49.16" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (144 samples, 0.04%)</title><rect x="1066.0" y="549" width="0.5" height="15.0" fill="rgb(215,15,47)" rx="2" ry="2" />
<text  x="1069.01" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).handlePacket (14,148 samples, 4.34%)</title><rect x="727.2" y="725" width="51.1" height="15.0" fill="rgb(246,47,31)" rx="2" ry="2" />
<text  x="730.17" y="735.5" >gviso..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (132 samples, 0.04%)</title><rect x="722.9" y="725" width="0.5" height="15.0" fill="rgb(239,106,22)" rx="2" ry="2" />
<text  x="725.88" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (71 samples, 0.02%)</title><rect x="72.5" y="709" width="0.3" height="15.0" fill="rgb(238,137,34)" rx="2" ry="2" />
<text  x="75.50" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (161 samples, 0.05%)</title><rect x="1060.2" y="693" width="0.6" height="15.0" fill="rgb(242,82,33)" rx="2" ry="2" />
<text  x="1063.22" y="703.5" ></text>
</g>
<g >
<title>runtime.checkTimers (95 samples, 0.03%)</title><rect x="1059.7" y="741" width="0.3" height="15.0" fill="rgb(238,10,46)" rx="2" ry="2" />
<text  x="1062.66" y="751.5" ></text>
</g>
<g >
<title>runtime.casgstatus (40 samples, 0.01%)</title><rect x="92.2" y="661" width="0.1" height="15.0" fill="rgb(248,167,3)" rx="2" ry="2" />
<text  x="95.18" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (894 samples, 0.27%)</title><rect x="953.3" y="485" width="3.2" height="15.0" fill="rgb(242,207,29)" rx="2" ry="2" />
<text  x="956.31" y="495.5" ></text>
</g>
<g >
<title>runtime.systemstack (46 samples, 0.01%)</title><rect x="711.5" y="709" width="0.1" height="15.0" fill="rgb(227,188,32)" rx="2" ry="2" />
<text  x="714.46" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (120 samples, 0.04%)</title><rect x="157.4" y="485" width="0.4" height="15.0" fill="rgb(243,79,4)" rx="2" ry="2" />
<text  x="160.39" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*protocol).Parse (61 samples, 0.02%)</title><rect x="795.6" y="757" width="0.2" height="15.0" fill="rgb(252,188,37)" rx="2" ry="2" />
<text  x="798.56" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (78 samples, 0.02%)</title><rect x="637.7" y="629" width="0.3" height="15.0" fill="rgb(230,183,12)" rx="2" ry="2" />
<text  x="640.72" y="639.5" ></text>
</g>
<g >
<title>runtime.mallocgc (115 samples, 0.04%)</title><rect x="1086.5" y="709" width="0.4" height="15.0" fill="rgb(250,150,45)" rx="2" ry="2" />
<text  x="1089.49" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,856 samples, 0.57%)</title><rect x="194.7" y="421" width="6.7" height="15.0" fill="rgb(205,4,7)" rx="2" ry="2" />
<text  x="197.69" y="431.5" ></text>
</g>
<g >
<title>runtime.step (301 samples, 0.09%)</title><rect x="1141.0" y="677" width="1.1" height="15.0" fill="rgb(207,113,39)" rx="2" ry="2" />
<text  x="1144.02" y="687.5" ></text>
</g>
<g >
<title>runtime.findfunc (31 samples, 0.01%)</title><rect x="1014.1" y="709" width="0.2" height="15.0" fill="rgb(247,120,47)" rx="2" ry="2" />
<text  x="1017.14" y="719.5" ></text>
</g>
<g >
<title>runtime.schedule (38 samples, 0.01%)</title><rect x="54.5" y="661" width="0.2" height="15.0" fill="rgb(218,76,38)" rx="2" ry="2" />
<text  x="57.51" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,850 samples, 1.18%)</title><rect x="134.5" y="565" width="13.9" height="15.0" fill="rgb(252,134,10)" rx="2" ry="2" />
<text  x="137.49" y="575.5" ></text>
</g>
<g >
<title>runtime.mallocgc (51 samples, 0.02%)</title><rect x="215.4" y="565" width="0.2" height="15.0" fill="rgb(218,185,47)" rx="2" ry="2" />
<text  x="218.38" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (360 samples, 0.11%)</title><rect x="14.6" y="725" width="1.3" height="15.0" fill="rgb(225,63,51)" rx="2" ry="2" />
<text  x="17.56" y="735.5" ></text>
</g>
<g >
<title>runtime.runqgrab (80 samples, 0.02%)</title><rect x="1185.7" y="789" width="0.3" height="15.0" fill="rgb(217,72,3)" rx="2" ry="2" />
<text  x="1188.71" y="799.5" ></text>
</g>
<g >
<title>runtime.futex (5,240 samples, 1.61%)</title><rect x="129.5" y="597" width="18.9" height="15.0" fill="rgb(226,196,28)" rx="2" ry="2" />
<text  x="132.47" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).MainAddress.func1 (35 samples, 0.01%)</title><rect x="725.3" y="693" width="0.2" height="15.0" fill="rgb(216,123,37)" rx="2" ry="2" />
<text  x="728.33" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).getRegs (36 samples, 0.01%)</title><rect x="638.6" y="821" width="0.1" height="15.0" fill="rgb(250,151,11)" rx="2" ry="2" />
<text  x="641.56" y="831.5" ></text>
</g>
<g >
<title>runtime.findrunnable (396 samples, 0.12%)</title><rect x="21.9" y="821" width="1.4" height="15.0" fill="rgb(227,216,42)" rx="2" ry="2" />
<text  x="24.85" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,964 samples, 1.83%)</title><rect x="616.4" y="677" width="21.6" height="15.0" fill="rgb(218,27,50)" rx="2" ry="2" />
<text  x="619.43" y="687.5" >[..</text>
</g>
<g >
<title>runtime.ready (92 samples, 0.03%)</title><rect x="65.6" y="581" width="0.4" height="15.0" fill="rgb(238,103,4)" rx="2" ry="2" />
<text  x="68.64" y="591.5" ></text>
</g>
<g >
<title>runtime.newobject (73 samples, 0.02%)</title><rect x="50.0" y="661" width="0.2" height="15.0" fill="rgb(230,34,40)" rx="2" ry="2" />
<text  x="52.97" y="671.5" ></text>
</g>
<g >
<title>runtime.systemstack (41 samples, 0.01%)</title><rect x="1091.7" y="565" width="0.1" height="15.0" fill="rgb(247,87,52)" rx="2" ry="2" />
<text  x="1094.67" y="575.5" ></text>
</g>
<g >
<title>runtime.step (54 samples, 0.02%)</title><rect x="1104.3" y="677" width="0.2" height="15.0" fill="rgb(214,30,7)" rx="2" ry="2" />
<text  x="1107.27" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).Enabled (28 samples, 0.01%)</title><rect x="736.5" y="661" width="0.1" height="15.0" fill="rgb(212,199,4)" rx="2" ry="2" />
<text  x="739.55" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (103 samples, 0.03%)</title><rect x="13.0" y="677" width="0.4" height="15.0" fill="rgb(231,142,50)" rx="2" ry="2" />
<text  x="16.02" y="687.5" ></text>
</g>
<g >
<title>runtime.stackpoolfree (43 samples, 0.01%)</title><rect x="1153.8" y="709" width="0.2" height="15.0" fill="rgb(207,126,32)" rx="2" ry="2" />
<text  x="1156.80" y="719.5" ></text>
</g>
<g >
<title>runtime.mapiternext (110 samples, 0.03%)</title><rect x="790.1" y="677" width="0.4" height="15.0" fill="rgb(245,20,7)" rx="2" ry="2" />
<text  x="793.09" y="687.5" ></text>
</g>
<g >
<title>runtime.(*mheap).allocSpan (40 samples, 0.01%)</title><rect x="1042.4" y="645" width="0.2" height="15.0" fill="rgb(249,208,12)" rx="2" ry="2" />
<text  x="1045.44" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (30 samples, 0.01%)</title><rect x="56.9" y="629" width="0.1" height="15.0" fill="rgb(254,15,0)" rx="2" ry="2" />
<text  x="59.87" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.newReceiver (73 samples, 0.02%)</title><rect x="1008.8" y="773" width="0.2" height="15.0" fill="rgb(247,9,22)" rx="2" ry="2" />
<text  x="1011.75" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*handshake).processSegments (3,168 samples, 0.97%)</title><rect x="1008.0" y="837" width="11.4" height="15.0" fill="rgb(212,197,8)" rx="2" ry="2" />
<text  x="1010.99" y="847.5" ></text>
</g>
<g >
<title>runtime.step (128 samples, 0.04%)</title><rect x="1149.3" y="661" width="0.5" height="15.0" fill="rgb(235,89,27)" rx="2" ry="2" />
<text  x="1152.33" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/fd.(*ReadWriter).WriteAt (2,303 samples, 0.71%)</title><rect x="193.5" y="517" width="8.3" height="15.0" fill="rgb(224,86,8)" rx="2" ry="2" />
<text  x="196.49" y="527.5" ></text>
</g>
<g >
<title>[[vdso]] (618 samples, 0.19%)</title><rect x="10.5" y="869" width="2.2" height="15.0" fill="rgb(205,219,30)" rx="2" ry="2" />
<text  x="13.47" y="879.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.(*IPv4).SourceAddress (47 samples, 0.01%)</title><rect x="759.0" y="629" width="0.2" height="15.0" fill="rgb(249,48,48)" rx="2" ry="2" />
<text  x="762.00" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (171 samples, 0.05%)</title><rect x="1127.9" y="661" width="0.7" height="15.0" fill="rgb(212,13,38)" rx="2" ry="2" />
<text  x="1130.94" y="671.5" ></text>
</g>
<g >
<title>runtime.pcvalue (153 samples, 0.05%)</title><rect x="1150.1" y="709" width="0.5" height="15.0" fill="rgb(236,17,1)" rx="2" ry="2" />
<text  x="1153.08" y="719.5" ></text>
</g>
<g >
<title>runtime.mapassign_fast32 (57 samples, 0.02%)</title><rect x="1023.0" y="757" width="0.2" height="15.0" fill="rgb(248,198,41)" rx="2" ry="2" />
<text  x="1025.98" y="767.5" ></text>
</g>
<g >
<title>runtime.futex (61 samples, 0.02%)</title><rect x="1051.1" y="629" width="0.3" height="15.0" fill="rgb(238,103,6)" rx="2" ry="2" />
<text  x="1054.13" y="639.5" ></text>
</g>
<g >
<title>runtime.(*pageAlloc).alloc (31 samples, 0.01%)</title><rect x="1153.1" y="677" width="0.1" height="15.0" fill="rgb(222,195,45)" rx="2" ry="2" />
<text  x="1156.12" y="687.5" ></text>
</g>
<g >
<title>runtime.mallocgc (1,041 samples, 0.32%)</title><rect x="764.5" y="597" width="3.7" height="15.0" fill="rgb(211,209,0)" rx="2" ry="2" />
<text  x="767.47" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,437 samples, 0.75%)</title><rect x="629.2" y="661" width="8.8" height="15.0" fill="rgb(208,66,6)" rx="2" ry="2" />
<text  x="632.19" y="671.5" ></text>
</g>
<g >
<title>runtime.mapassign (104 samples, 0.03%)</title><rect x="1023.8" y="773" width="0.4" height="15.0" fill="rgb(209,176,35)" rx="2" ry="2" />
<text  x="1026.79" y="783.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (49 samples, 0.02%)</title><rect x="1012.8" y="693" width="0.2" height="15.0" fill="rgb(211,89,30)" rx="2" ry="2" />
<text  x="1015.82" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).maxOptionSize (112 samples, 0.03%)</title><rect x="1010.0" y="757" width="0.4" height="15.0" fill="rgb(220,211,35)" rx="2" ry="2" />
<text  x="1012.98" y="767.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (55 samples, 0.02%)</title><rect x="713.6" y="789" width="0.2" height="15.0" fill="rgb(226,46,48)" rx="2" ry="2" />
<text  x="716.62" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*SocketOperations).Readiness (144 samples, 0.04%)</title><rect x="169.5" y="741" width="0.5" height="15.0" fill="rgb(228,184,31)" rx="2" ry="2" />
<text  x="172.46" y="751.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (70 samples, 0.02%)</title><rect x="1067.5" y="773" width="0.3" height="15.0" fill="rgb(231,11,2)" rx="2" ry="2" />
<text  x="1070.53" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doStop (29 samples, 0.01%)</title><rect x="33.0" y="869" width="0.1" height="15.0" fill="rgb(226,215,7)" rx="2" ry="2" />
<text  x="35.96" y="879.5" ></text>
</g>
<g >
<title>runtime.mcall (18,524 samples, 5.68%)</title><rect x="91.7" y="693" width="67.0" height="15.0" fill="rgb(218,79,45)" rx="2" ry="2" />
<text  x="94.68" y="703.5" >runtime..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (146 samples, 0.04%)</title><rect x="702.3" y="693" width="0.6" height="15.0" fill="rgb(229,14,7)" rx="2" ry="2" />
<text  x="705.33" y="703.5" ></text>
</g>
<g >
<title>runtime.newobject (70 samples, 0.02%)</title><rect x="1055.7" y="789" width="0.3" height="15.0" fill="rgb(246,62,31)" rx="2" ry="2" />
<text  x="1058.71" y="799.5" ></text>
</g>
<g >
<title>runtime.osyield (55 samples, 0.02%)</title><rect x="1151.0" y="709" width="0.2" height="15.0" fill="rgb(207,161,39)" rx="2" ry="2" />
<text  x="1154.03" y="719.5" ></text>
</g>
<g >
<title>runtime.(*stackScanState).getPtr (181 samples, 0.06%)</title><rect x="1133.1" y="773" width="0.7" height="15.0" fill="rgb(248,5,23)" rx="2" ry="2" />
<text  x="1136.12" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (34 samples, 0.01%)</title><rect x="718.3" y="805" width="0.1" height="15.0" fill="rgb(244,11,32)" rx="2" ry="2" />
<text  x="721.33" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).resolvedFields (218 samples, 0.07%)</title><rect x="1074.5" y="629" width="0.7" height="15.0" fill="rgb(243,25,13)" rx="2" ry="2" />
<text  x="1077.46" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (37 samples, 0.01%)</title><rect x="1006.0" y="645" width="0.1" height="15.0" fill="rgb(214,84,34)" rx="2" ry="2" />
<text  x="1008.95" y="655.5" ></text>
</g>
<g >
<title>runtime.futex (634 samples, 0.19%)</title><rect x="210.8" y="293" width="2.3" height="15.0" fill="rgb(223,175,31)" rx="2" ry="2" />
<text  x="213.82" y="303.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (189 samples, 0.06%)</title><rect x="1130.2" y="725" width="0.7" height="15.0" fill="rgb(231,31,11)" rx="2" ry="2" />
<text  x="1133.22" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (29 samples, 0.01%)</title><rect x="1035.0" y="693" width="0.1" height="15.0" fill="rgb(253,13,30)" rx="2" ry="2" />
<text  x="1037.98" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,157 samples, 0.66%)</title><rect x="193.6" y="485" width="7.8" height="15.0" fill="rgb(221,81,3)" rx="2" ry="2" />
<text  x="196.60" y="495.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (64 samples, 0.02%)</title><rect x="1028.5" y="389" width="0.3" height="15.0" fill="rgb(212,20,24)" rx="2" ry="2" />
<text  x="1031.54" y="399.5" ></text>
</g>
<g >
<title>runtime.findfunc (77 samples, 0.02%)</title><rect x="1112.4" y="821" width="0.3" height="15.0" fill="rgb(219,20,52)" rx="2" ry="2" />
<text  x="1115.42" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (641 samples, 0.20%)</title><rect x="908.5" y="645" width="2.3" height="15.0" fill="rgb(238,177,0)" rx="2" ry="2" />
<text  x="911.46" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (3,688 samples, 1.13%)</title><rect x="745.5" y="629" width="13.3" height="15.0" fill="rgb(216,87,42)" rx="2" ry="2" />
<text  x="748.51" y="639.5" ></text>
</g>
<g >
<title>runtime.newobject (36 samples, 0.01%)</title><rect x="1023.5" y="757" width="0.1" height="15.0" fill="rgb(216,107,41)" rx="2" ry="2" />
<text  x="1026.47" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (675 samples, 0.21%)</title><rect x="916.3" y="549" width="2.4" height="15.0" fill="rgb(254,170,42)" rx="2" ry="2" />
<text  x="919.29" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isValidForOutgoingRLocked (70 samples, 0.02%)</title><rect x="214.6" y="549" width="0.3" height="15.0" fill="rgb(215,22,16)" rx="2" ry="2" />
<text  x="217.62" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).DeliverOutboundPacket (52 samples, 0.02%)</title><rect x="1073.8" y="549" width="0.2" height="15.0" fill="rgb(249,212,14)" rx="2" ry="2" />
<text  x="1076.81" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).getAddressOrCreateTemp (38 samples, 0.01%)</title><rect x="795.2" y="757" width="0.2" height="15.0" fill="rgb(237,25,38)" rx="2" ry="2" />
<text  x="798.22" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*timer).init (235 samples, 0.07%)</title><rect x="1010.6" y="757" width="0.9" height="15.0" fill="rgb(224,69,33)" rx="2" ry="2" />
<text  x="1013.64" y="767.5" ></text>
</g>
<g >
<title>runtime.park_m (326 samples, 0.10%)</title><rect x="17.5" y="837" width="1.2" height="15.0" fill="rgb(217,137,53)" rx="2" ry="2" />
<text  x="20.50" y="847.5" ></text>
</g>
<g >
<title>runtime.convI2I (33 samples, 0.01%)</title><rect x="203.1" y="725" width="0.1" height="15.0" fill="rgb(220,105,47)" rx="2" ry="2" />
<text  x="206.08" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (245 samples, 0.08%)</title><rect x="1065.6" y="629" width="0.9" height="15.0" fill="rgb(233,9,5)" rx="2" ry="2" />
<text  x="1068.64" y="639.5" ></text>
</g>
<g >
<title>runtime.newobject (48 samples, 0.01%)</title><rect x="181.8" y="469" width="0.2" height="15.0" fill="rgb(228,88,39)" rx="2" ry="2" />
<text  x="184.84" y="479.5" ></text>
</g>
<g >
<title>runtime.newobject (38 samples, 0.01%)</title><rect x="1019.6" y="805" width="0.1" height="15.0" fill="rgb(232,136,35)" rx="2" ry="2" />
<text  x="1022.58" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).handleUpperLevelConfirmation (2,578 samples, 0.79%)</title><rect x="1046.9" y="837" width="9.3" height="15.0" fill="rgb(231,121,4)" rx="2" ry="2" />
<text  x="1049.88" y="847.5" ></text>
</g>
<g >
<title>runtime.lock2 (41 samples, 0.01%)</title><rect x="112.5" y="613" width="0.2" height="15.0" fill="rgb(238,108,26)" rx="2" ry="2" />
<text  x="115.53" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (141 samples, 0.04%)</title><rect x="921.4" y="533" width="0.5" height="15.0" fill="rgb(249,19,35)" rx="2" ry="2" />
<text  x="924.36" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IncRef (118 samples, 0.04%)</title><rect x="735.9" y="677" width="0.5" height="15.0" fill="rgb(212,168,25)" rx="2" ry="2" />
<text  x="738.93" y="687.5" ></text>
</g>
<g >
<title>runtime.stopm (37 samples, 0.01%)</title><rect x="157.8" y="645" width="0.2" height="15.0" fill="rgb(238,163,19)" rx="2" ry="2" />
<text  x="160.82" y="655.5" ></text>
</g>
<g >
<title>runtime.futex (69 samples, 0.02%)</title><rect x="17.9" y="757" width="0.3" height="15.0" fill="rgb(222,105,23)" rx="2" ry="2" />
<text  x="20.93" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/goid.getg (53 samples, 0.02%)</title><rect x="83.0" y="693" width="0.2" height="15.0" fill="rgb(252,66,25)" rx="2" ry="2" />
<text  x="86.01" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBuffer).consume (117 samples, 0.04%)</title><rect x="845.1" y="821" width="0.4" height="15.0" fill="rgb(218,193,39)" rx="2" ry="2" />
<text  x="848.11" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="1173.1" y="789" width="0.1" height="15.0" fill="rgb(235,213,32)" rx="2" ry="2" />
<text  x="1176.09" y="799.5" ></text>
</g>
<g >
<title>runtime.mstart (47 samples, 0.01%)</title><rect x="16.8" y="757" width="0.1" height="15.0" fill="rgb(219,211,41)" rx="2" ry="2" />
<text  x="19.75" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).entry (71 samples, 0.02%)</title><rect x="213.7" y="469" width="0.3" height="15.0" fill="rgb(233,66,45)" rx="2" ry="2" />
<text  x="216.71" y="479.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (123 samples, 0.04%)</title><rect x="904.5" y="693" width="0.4" height="15.0" fill="rgb(235,119,16)" rx="2" ry="2" />
<text  x="907.46" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).GSOMaxSize (114 samples, 0.03%)</title><rect x="1038.2" y="757" width="0.4" height="15.0" fill="rgb(224,160,47)" rx="2" ry="2" />
<text  x="1041.21" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (102 samples, 0.03%)</title><rect x="701.2" y="597" width="0.4" height="15.0" fill="rgb(221,218,43)" rx="2" ry="2" />
<text  x="704.22" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (66 samples, 0.02%)</title><rect x="788.1" y="645" width="0.3" height="15.0" fill="rgb(228,201,16)" rx="2" ry="2" />
<text  x="791.12" y="655.5" ></text>
</g>
<g >
<title>runtime.sellock (56 samples, 0.02%)</title><rect x="161.6" y="709" width="0.2" height="15.0" fill="rgb(216,165,5)" rx="2" ry="2" />
<text  x="164.60" y="719.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (66 samples, 0.02%)</title><rect x="72.0" y="725" width="0.2" height="15.0" fill="rgb(205,0,32)" rx="2" ry="2" />
<text  x="74.99" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.sendTCP (1,935 samples, 0.59%)</title><rect x="208.6" y="597" width="7.0" height="15.0" fill="rgb(208,179,25)" rx="2" ry="2" />
<text  x="211.57" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).handlePacket (92 samples, 0.03%)</title><rect x="19.7" y="821" width="0.3" height="15.0" fill="rgb(223,167,54)" rx="2" ry="2" />
<text  x="22.66" y="831.5" ></text>
</g>
<g >
<title>runtime.stackfree (129 samples, 0.04%)</title><rect x="1153.5" y="741" width="0.5" height="15.0" fill="rgb(211,173,13)" rx="2" ry="2" />
<text  x="1156.52" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (117 samples, 0.04%)</title><rect x="732.4" y="693" width="0.4" height="15.0" fill="rgb(243,101,43)" rx="2" ry="2" />
<text  x="735.36" y="703.5" ></text>
</g>
<g >
<title>runtime.stopm (110 samples, 0.03%)</title><rect x="701.6" y="773" width="0.4" height="15.0" fill="rgb(209,168,29)" rx="2" ry="2" />
<text  x="704.60" y="783.5" ></text>
</g>
<g >
<title>runtime.runqsteal (70 samples, 0.02%)</title><rect x="157.1" y="645" width="0.2" height="15.0" fill="rgb(205,82,1)" rx="2" ry="2" />
<text  x="160.08" y="655.5" ></text>
</g>
<g >
<title>io.ReadAtLeast (55 samples, 0.02%)</title><rect x="1041.2" y="741" width="0.2" height="15.0" fill="rgb(247,147,27)" rx="2" ry="2" />
<text  x="1044.22" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut.func1 (32 samples, 0.01%)</title><rect x="57.2" y="645" width="0.1" height="15.0" fill="rgb(247,163,11)" rx="2" ry="2" />
<text  x="60.22" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (54 samples, 0.02%)</title><rect x="1130.7" y="645" width="0.2" height="15.0" fill="rgb(245,64,47)" rx="2" ry="2" />
<text  x="1133.71" y="655.5" ></text>
</g>
<g >
<title>runtime.startm (127 samples, 0.04%)</title><rect x="18.2" y="773" width="0.5" height="15.0" fill="rgb(236,38,43)" rx="2" ry="2" />
<text  x="21.21" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).FloatingPointData (80 samples, 0.02%)</title><rect x="222.8" y="821" width="0.3" height="15.0" fill="rgb(223,130,30)" rx="2" ry="2" />
<text  x="225.83" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/packetsocket.(*endpoint).WritePacket (331 samples, 0.10%)</title><rect x="1027.8" y="613" width="1.2" height="15.0" fill="rgb(226,136,16)" rx="2" ry="2" />
<text  x="1030.79" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (187 samples, 0.06%)</title><rect x="702.2" y="709" width="0.7" height="15.0" fill="rgb(232,224,37)" rx="2" ry="2" />
<text  x="705.18" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (120 samples, 0.04%)</title><rect x="157.4" y="517" width="0.4" height="15.0" fill="rgb(240,173,20)" rx="2" ry="2" />
<text  x="160.39" y="527.5" ></text>
</g>
<g >
<title>time.Now (36 samples, 0.01%)</title><rect x="768.8" y="629" width="0.1" height="15.0" fill="rgb(229,59,21)" rx="2" ry="2" />
<text  x="771.76" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.RecvFrom (1,617 samples, 0.50%)</title><rect x="173.5" y="773" width="5.9" height="15.0" fill="rgb(209,46,35)" rx="2" ry="2" />
<text  x="176.50" y="783.5" ></text>
</g>
<g >
<title>runtime.siftdownTimer (33 samples, 0.01%)</title><rect x="1069.1" y="757" width="0.2" height="15.0" fill="rgb(253,214,17)" rx="2" ry="2" />
<text  x="1072.14" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.newOutgoingSegment (99 samples, 0.03%)</title><rect x="183.1" y="709" width="0.3" height="15.0" fill="rgb(218,128,53)" rx="2" ry="2" />
<text  x="186.07" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (40 samples, 0.01%)</title><rect x="1093.2" y="693" width="0.2" height="15.0" fill="rgb(216,67,13)" rx="2" ry="2" />
<text  x="1096.21" y="703.5" ></text>
</g>
<g >
<title>runtime.mallocgc (46 samples, 0.01%)</title><rect x="1008.8" y="741" width="0.2" height="15.0" fill="rgb(254,138,19)" rx="2" ry="2" />
<text  x="1011.82" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.FromIOWriter.WriteFromBlocks (2,394 samples, 0.73%)</title><rect x="193.2" y="565" width="8.7" height="15.0" fill="rgb(246,87,21)" rx="2" ry="2" />
<text  x="196.20" y="575.5" ></text>
</g>
<g >
<title>runtime.chanrecv2 (432 samples, 0.13%)</title><rect x="705.1" y="853" width="1.6" height="15.0" fill="rgb(247,113,21)" rx="2" ry="2" />
<text  x="708.15" y="863.5" ></text>
</g>
<g >
<title>runtime.LockOSThread (42 samples, 0.01%)</title><rect x="639.6" y="821" width="0.2" height="15.0" fill="rgb(217,36,15)" rx="2" ry="2" />
<text  x="642.62" y="831.5" ></text>
</g>
<g >
<title>ext4_mark_iloc_dirty (142 samples, 0.04%)</title><rect x="200.2" y="229" width="0.5" height="15.0" fill="rgb(214,58,12)" rx="2" ry="2" />
<text  x="203.18" y="239.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (38 samples, 0.01%)</title><rect x="1028.6" y="325" width="0.2" height="15.0" fill="rgb(226,20,8)" rx="2" ry="2" />
<text  x="1031.64" y="335.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).Fetch (133 samples, 0.04%)</title><rect x="1021.4" y="853" width="0.5" height="15.0" fill="rgb(226,30,13)" rx="2" ry="2" />
<text  x="1024.43" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (85 samples, 0.03%)</title><rect x="23.0" y="709" width="0.3" height="15.0" fill="rgb(230,96,53)" rx="2" ry="2" />
<text  x="25.96" y="719.5" ></text>
</g>
<g >
<title>runtime.mcall (1,356 samples, 0.42%)</title><rect x="1185.1" y="869" width="4.9" height="15.0" fill="rgb(216,116,46)" rx="2" ry="2" />
<text  x="1188.10" y="879.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (113 samples, 0.03%)</title><rect x="203.9" y="693" width="0.4" height="15.0" fill="rgb(242,141,4)" rx="2" ry="2" />
<text  x="206.88" y="703.5" ></text>
</g>
<g >
<title>runtime.step (31 samples, 0.01%)</title><rect x="1184.9" y="837" width="0.1" height="15.0" fill="rgb(251,123,29)" rx="2" ry="2" />
<text  x="1187.86" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).dumpPacket (55 samples, 0.02%)</title><rect x="796.4" y="789" width="0.2" height="15.0" fill="rgb(237,145,14)" rx="2" ry="2" />
<text  x="799.45" y="799.5" ></text>
</g>
<g >
<title>runtime.wakep (90 samples, 0.03%)</title><rect x="19.7" y="645" width="0.3" height="15.0" fill="rgb(240,180,2)" rx="2" ry="2" />
<text  x="22.66" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*segment).setOwner (42 samples, 0.01%)</title><rect x="760.2" y="597" width="0.2" height="15.0" fill="rgb(212,71,4)" rx="2" ry="2" />
<text  x="763.23" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (358 samples, 0.11%)</title><rect x="995.3" y="469" width="1.3" height="15.0" fill="rgb(234,87,53)" rx="2" ry="2" />
<text  x="998.34" y="479.5" ></text>
</g>
<g >
<title>runtime.newobject (107 samples, 0.03%)</title><rect x="66.5" y="693" width="0.4" height="15.0" fill="rgb(246,189,11)" rx="2" ry="2" />
<text  x="69.53" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,124 samples, 1.57%)</title><rect x="129.9" y="581" width="18.5" height="15.0" fill="rgb(238,71,52)" rx="2" ry="2" />
<text  x="132.89" y="591.5" ></text>
</g>
<g >
<title>runtime.mallocgc (29 samples, 0.01%)</title><rect x="1010.7" y="725" width="0.1" height="15.0" fill="rgb(245,32,26)" rx="2" ry="2" />
<text  x="1013.69" y="735.5" ></text>
</g>
<g >
<title>runtime.bgscavenge.func2 (247 samples, 0.08%)</title><rect x="1130.1" y="853" width="0.9" height="15.0" fill="rgb(228,143,4)" rx="2" ry="2" />
<text  x="1133.07" y="863.5" ></text>
</g>
<g >
<title>runtime.mallocgc (108 samples, 0.03%)</title><rect x="171.5" y="725" width="0.4" height="15.0" fill="rgb(244,35,12)" rx="2" ry="2" />
<text  x="174.48" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (65 samples, 0.02%)</title><rect x="1073.8" y="565" width="0.2" height="15.0" fill="rgb(254,51,28)" rx="2" ry="2" />
<text  x="1076.76" y="575.5" ></text>
</g>
<g >
<title>runtime.sellock (442 samples, 0.14%)</title><rect x="159.2" y="693" width="1.6" height="15.0" fill="rgb(215,157,36)" rx="2" ry="2" />
<text  x="162.18" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Writev (29 samples, 0.01%)</title><rect x="219.9" y="789" width="0.1" height="15.0" fill="rgb(215,207,8)" rx="2" ry="2" />
<text  x="222.94" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (35 samples, 0.01%)</title><rect x="1079.1" y="629" width="0.1" height="15.0" fill="rgb(216,211,48)" rx="2" ry="2" />
<text  x="1082.08" y="639.5" ></text>
</g>
<g >
<title>runtime.LockOSThread (68 samples, 0.02%)</title><rect x="454.1" y="805" width="0.2" height="15.0" fill="rgb(246,4,29)" rx="2" ry="2" />
<text  x="457.08" y="815.5" ></text>
</g>
<g >
<title>runtime.findfunc (64 samples, 0.02%)</title><rect x="1016.6" y="725" width="0.2" height="15.0" fill="rgb(215,201,48)" rx="2" ry="2" />
<text  x="1019.58" y="735.5" ></text>
</g>
<g >
<title>time.AfterFunc (72 samples, 0.02%)</title><rect x="1019.6" y="821" width="0.2" height="15.0" fill="rgb(251,228,11)" rx="2" ry="2" />
<text  x="1022.57" y="831.5" ></text>
</g>
<g >
<title>runtime.convI2I (35 samples, 0.01%)</title><rect x="216.6" y="725" width="0.2" height="15.0" fill="rgb(221,181,4)" rx="2" ry="2" />
<text  x="219.65" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (282 samples, 0.09%)</title><rect x="703.9" y="757" width="1.0" height="15.0" fill="rgb(219,95,26)" rx="2" ry="2" />
<text  x="706.86" y="767.5" ></text>
</g>
<g >
<title>runtime.newstack (897 samples, 0.27%)</title><rect x="1101.9" y="757" width="3.2" height="15.0" fill="rgb(249,166,31)" rx="2" ry="2" />
<text  x="1104.87" y="767.5" ></text>
</g>
<g >
<title>runtime.mallocgc (63 samples, 0.02%)</title><rect x="206.8" y="645" width="0.2" height="15.0" fill="rgb(222,20,6)" rx="2" ry="2" />
<text  x="209.80" y="655.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (44 samples, 0.01%)</title><rect x="746.2" y="597" width="0.2" height="15.0" fill="rgb(219,38,53)" rx="2" ry="2" />
<text  x="749.20" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/rand.Read (67 samples, 0.02%)</title><rect x="1041.2" y="757" width="0.2" height="15.0" fill="rgb(230,53,10)" rx="2" ry="2" />
<text  x="1044.18" y="767.5" ></text>
</g>
<g >
<title>runtime.memhash64 (54 samples, 0.02%)</title><rect x="775.0" y="677" width="0.1" height="15.0" fill="rgb(239,164,43)" rx="2" ry="2" />
<text  x="777.95" y="687.5" ></text>
</g>
<g >
<title>runtime.step (49 samples, 0.02%)</title><rect x="1142.1" y="693" width="0.2" height="15.0" fill="rgb(211,149,37)" rx="2" ry="2" />
<text  x="1145.11" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).WritePacket (468 samples, 0.14%)</title><rect x="1091.1" y="709" width="1.7" height="15.0" fill="rgb(237,2,32)" rx="2" ry="2" />
<text  x="1094.10" y="719.5" ></text>
</g>
<g >
<title>runtime.ready (90 samples, 0.03%)</title><rect x="19.7" y="661" width="0.3" height="15.0" fill="rgb(228,64,52)" rx="2" ry="2" />
<text  x="22.66" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/rawfile.BlockingPoll (6,957 samples, 2.13%)</title><rect x="797.0" y="821" width="25.2" height="15.0" fill="rgb(230,90,18)" rx="2" ry="2" />
<text  x="800.01" y="831.5" >g..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (36 samples, 0.01%)</title><rect x="1051.6" y="693" width="0.2" height="15.0" fill="rgb(243,200,49)" rx="2" ry="2" />
<text  x="1054.64" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBuffer).consume (45 samples, 0.01%)</title><rect x="780.9" y="709" width="0.1" height="15.0" fill="rgb(221,101,18)" rx="2" ry="2" />
<text  x="783.86" y="719.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (77 samples, 0.02%)</title><rect x="67.8" y="693" width="0.3" height="15.0" fill="rgb(245,192,17)" rx="2" ry="2" />
<text  x="70.79" y="703.5" ></text>
</g>
<g >
<title>runtime.stackpoolalloc (74 samples, 0.02%)</title><rect x="1018.0" y="709" width="0.2" height="15.0" fill="rgb(246,222,44)" rx="2" ry="2" />
<text  x="1020.98" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (83 samples, 0.03%)</title><rect x="1085.8" y="645" width="0.3" height="15.0" fill="rgb(254,161,7)" rx="2" ry="2" />
<text  x="1088.84" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (47 samples, 0.01%)</title><rect x="1005.9" y="677" width="0.2" height="15.0" fill="rgb(216,141,21)" rx="2" ry="2" />
<text  x="1008.92" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).NowNanoseconds (37 samples, 0.01%)</title><rect x="1047.2" y="789" width="0.1" height="15.0" fill="rgb(206,169,35)" rx="2" ry="2" />
<text  x="1050.20" y="799.5" ></text>
</g>
<g >
<title>runtime.ready (76 samples, 0.02%)</title><rect x="1051.1" y="693" width="0.3" height="15.0" fill="rgb(226,77,39)" rx="2" ry="2" />
<text  x="1054.08" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (16,543 samples, 5.07%)</title><rect x="938.3" y="693" width="59.8" height="15.0" fill="rgb(232,109,51)" rx="2" ry="2" />
<text  x="941.30" y="703.5" >[[kern..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Readiness (286 samples, 0.09%)</title><rect x="165.7" y="709" width="1.0" height="15.0" fill="rgb(235,82,23)" rx="2" ry="2" />
<text  x="168.68" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).TransportProtocolOption (91 samples, 0.03%)</title><rect x="1009.6" y="757" width="0.4" height="15.0" fill="rgb(225,97,38)" rx="2" ry="2" />
<text  x="1012.63" y="767.5" ></text>
</g>
<g >
<title>ext4_xattr_ibody_get (167 samples, 0.05%)</title><rect x="197.2" y="213" width="0.6" height="15.0" fill="rgb(212,137,19)" rx="2" ry="2" />
<text  x="200.18" y="223.5" ></text>
</g>
<g >
<title>runtime.convT2Enoptr (88 samples, 0.03%)</title><rect x="927.5" y="805" width="0.4" height="15.0" fill="rgb(221,42,9)" rx="2" ry="2" />
<text  x="930.53" y="815.5" ></text>
</g>
<g >
<title>runtime.selectnbsend (81 samples, 0.02%)</title><rect x="1071.6" y="693" width="0.2" height="15.0" fill="rgb(225,95,36)" rx="2" ry="2" />
<text  x="1074.56" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,296 samples, 0.40%)</title><rect x="914.0" y="661" width="4.7" height="15.0" fill="rgb(224,99,26)" rx="2" ry="2" />
<text  x="917.04" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).Done (43 samples, 0.01%)</title><rect x="1045.5" y="853" width="0.1" height="15.0" fill="rgb(213,106,30)" rx="2" ry="2" />
<text  x="1048.48" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (64 samples, 0.02%)</title><rect x="735.9" y="661" width="0.3" height="15.0" fill="rgb(207,74,21)" rx="2" ry="2" />
<text  x="738.94" y="671.5" ></text>
</g>
<g >
<title>runtime.notewakeup (123 samples, 0.04%)</title><rect x="1028.3" y="469" width="0.5" height="15.0" fill="rgb(218,64,32)" rx="2" ry="2" />
<text  x="1031.33" y="479.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (106 samples, 0.03%)</title><rect x="1119.3" y="661" width="0.4" height="15.0" fill="rgb(232,71,10)" rx="2" ry="2" />
<text  x="1122.34" y="671.5" ></text>
</g>
<g >
<title>runtime.epollwait (222 samples, 0.07%)</title><rect x="1060.0" y="741" width="0.8" height="15.0" fill="rgb(212,78,30)" rx="2" ry="2" />
<text  x="1063.00" y="751.5" ></text>
</g>
<g >
<title>runtime.selectgo (20,676 samples, 6.34%)</title><rect x="86.6" y="709" width="74.8" height="15.0" fill="rgb(245,10,20)" rx="2" ry="2" />
<text  x="89.58" y="719.5" >runtime...</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn (157 samples, 0.05%)</title><rect x="183.9" y="709" width="0.5" height="15.0" fill="rgb(238,62,45)" rx="2" ry="2" />
<text  x="186.86" y="719.5" ></text>
</g>
<g >
<title>time.resetTimer (29 samples, 0.01%)</title><rect x="1088.6" y="757" width="0.1" height="15.0" fill="rgb(220,113,37)" rx="2" ry="2" />
<text  x="1091.63" y="767.5" ></text>
</g>
<g >
<title>runtime.runqempty (33 samples, 0.01%)</title><rect x="841.7" y="773" width="0.1" height="15.0" fill="rgb(219,217,21)" rx="2" ry="2" />
<text  x="844.68" y="783.5" ></text>
</g>
<g >
<title>runtime.notesleep (161 samples, 0.05%)</title><rect x="451.7" y="677" width="0.5" height="15.0" fill="rgb(219,5,19)" rx="2" ry="2" />
<text  x="454.65" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (66 samples, 0.02%)</title><rect x="1189.8" y="709" width="0.2" height="15.0" fill="rgb(211,23,38)" rx="2" ry="2" />
<text  x="1192.75" y="719.5" ></text>
</g>
<g >
<title>runtime.sweepone (437 samples, 0.13%)</title><rect x="1109.4" y="853" width="1.6" height="15.0" fill="rgb(251,99,54)" rx="2" ry="2" />
<text  x="1112.43" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/procid.Current (63 samples, 0.02%)</title><rect x="222.4" y="821" width="0.2" height="15.0" fill="rgb(228,172,48)" rx="2" ry="2" />
<text  x="225.37" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (43 samples, 0.01%)</title><rect x="776.9" y="661" width="0.1" height="15.0" fill="rgb(221,180,10)" rx="2" ry="2" />
<text  x="779.85" y="671.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (56 samples, 0.02%)</title><rect x="1011.7" y="725" width="0.2" height="15.0" fill="rgb(206,167,41)" rx="2" ry="2" />
<text  x="1014.71" y="735.5" ></text>
</g>
<g >
<title>[unknown] (1,881 samples, 0.58%)</title><rect x="12.7" y="869" width="6.8" height="15.0" fill="rgb(213,0,18)" rx="2" ry="2" />
<text  x="15.70" y="879.5" ></text>
</g>
<g >
<title>br_dev_xmit (383 samples, 0.12%)</title><rect x="989.7" y="165" width="1.4" height="15.0" fill="rgb(249,173,38)" rx="2" ry="2" />
<text  x="992.68" y="175.5" ></text>
</g>
<g >
<title>runtime.adjustdefers (142 samples, 0.04%)</title><rect x="1013.8" y="741" width="0.5" height="15.0" fill="rgb(208,78,22)" rx="2" ry="2" />
<text  x="1016.76" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).MaxHeaderLength (61 samples, 0.02%)</title><rect x="209.0" y="565" width="0.3" height="15.0" fill="rgb(215,44,38)" rx="2" ry="2" />
<text  x="212.05" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,674 samples, 0.82%)</title><rect x="138.7" y="485" width="9.7" height="15.0" fill="rgb(216,30,50)" rx="2" ry="2" />
<text  x="141.75" y="495.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,959 samples, 0.60%)</title><rect x="985.3" y="197" width="7.0" height="15.0" fill="rgb(229,152,10)" rx="2" ry="2" />
<text  x="988.25" y="207.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).maybeSendSegment (829 samples, 0.25%)</title><rect x="180.0" y="661" width="3.0" height="15.0" fill="rgb(234,133,7)" rx="2" ry="2" />
<text  x="183.00" y="671.5" ></text>
</g>
<g >
<title>get_l4proto (63 samples, 0.02%)</title><rect x="961.4" y="453" width="0.3" height="15.0" fill="rgb(245,11,46)" rx="2" ry="2" />
<text  x="964.45" y="463.5" ></text>
</g>
<g >
<title>runtime.mallocgc (28 samples, 0.01%)</title><rect x="1025.1" y="757" width="0.1" height="15.0" fill="rgb(213,12,9)" rx="2" ry="2" />
<text  x="1028.09" y="767.5" ></text>
</g>
<g >
<title>runtime.makechan (65 samples, 0.02%)</title><rect x="1048.5" y="693" width="0.2" height="15.0" fill="rgb(238,10,4)" rx="2" ry="2" />
<text  x="1051.45" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (6,755 samples, 2.07%)</title><rect x="870.5" y="741" width="24.4" height="15.0" fill="rgb(211,214,10)" rx="2" ry="2" />
<text  x="873.46" y="751.5" >[..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="446.5" y="549" width="0.1" height="15.0" fill="rgb(217,215,4)" rx="2" ry="2" />
<text  x="449.50" y="559.5" ></text>
</g>
<g >
<title>runtime.procyield (312 samples, 0.10%)</title><rect x="1151.2" y="709" width="1.2" height="15.0" fill="rgb(254,161,15)" rx="2" ry="2" />
<text  x="1154.23" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (72 samples, 0.02%)</title><rect x="18.4" y="661" width="0.3" height="15.0" fill="rgb(205,168,12)" rx="2" ry="2" />
<text  x="21.40" y="671.5" ></text>
</g>
<g >
<title>runtime.notewakeup (191 samples, 0.06%)</title><rect x="702.2" y="741" width="0.7" height="15.0" fill="rgb(238,61,27)" rx="2" ry="2" />
<text  x="705.17" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,977 samples, 0.61%)</title><rect x="834.1" y="613" width="7.1" height="15.0" fill="rgb(222,191,28)" rx="2" ry="2" />
<text  x="837.10" y="623.5" ></text>
</g>
<g >
<title>runtime.systemstack (28 samples, 0.01%)</title><rect x="1068.3" y="789" width="0.1" height="15.0" fill="rgb(229,20,42)" rx="2" ry="2" />
<text  x="1071.34" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (51 samples, 0.02%)</title><rect x="1049.6" y="453" width="0.1" height="15.0" fill="rgb(222,213,22)" rx="2" ry="2" />
<text  x="1052.56" y="463.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (103 samples, 0.03%)</title><rect x="22.9" y="741" width="0.4" height="15.0" fill="rgb(225,84,23)" rx="2" ry="2" />
<text  x="25.90" y="751.5" ></text>
</g>
<g >
<title>runtime/trace.StartRegion (53 samples, 0.02%)</title><rect x="696.6" y="853" width="0.2" height="15.0" fill="rgb(240,70,40)" rx="2" ry="2" />
<text  x="699.61" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (44 samples, 0.01%)</title><rect x="704.9" y="789" width="0.1" height="15.0" fill="rgb(254,170,5)" rx="2" ry="2" />
<text  x="707.88" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (75 samples, 0.02%)</title><rect x="117.8" y="565" width="0.3" height="15.0" fill="rgb(233,69,37)" rx="2" ry="2" />
<text  x="120.82" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).AddressSpace (68 samples, 0.02%)</title><rect x="221.3" y="837" width="0.2" height="15.0" fill="rgb(233,5,38)" rx="2" ry="2" />
<text  x="224.29" y="847.5" ></text>
</g>
<g >
<title>runtime.(*mheap).freeSpan.func1 (134 samples, 0.04%)</title><rect x="1110.5" y="805" width="0.5" height="15.0" fill="rgb(235,156,47)" rx="2" ry="2" />
<text  x="1113.50" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.calculateAdvertisedMSS (28 samples, 0.01%)</title><rect x="1024.6" y="757" width="0.1" height="15.0" fill="rgb(243,196,12)" rx="2" ry="2" />
<text  x="1027.63" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBufferList).Remove (38 samples, 0.01%)</title><rect x="999.4" y="853" width="0.2" height="15.0" fill="rgb(207,132,36)" rx="2" ry="2" />
<text  x="1002.43" y="863.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (271 samples, 0.08%)</title><rect x="847.5" y="741" width="1.0" height="15.0" fill="rgb(214,223,36)" rx="2" ry="2" />
<text  x="850.47" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress.func1 (44 samples, 0.01%)</title><rect x="783.5" y="693" width="0.2" height="15.0" fill="rgb(214,133,48)" rx="2" ry="2" />
<text  x="786.51" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (29 samples, 0.01%)</title><rect x="149.2" y="421" width="0.2" height="15.0" fill="rgb(207,124,46)" rx="2" ry="2" />
<text  x="152.25" y="431.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (54 samples, 0.02%)</title><rect x="793.3" y="709" width="0.2" height="15.0" fill="rgb(208,30,13)" rx="2" ry="2" />
<text  x="796.31" y="719.5" ></text>
</g>
<g >
<title>runtime.newobject (392 samples, 0.12%)</title><rect x="1041.6" y="773" width="1.4" height="15.0" fill="rgb(210,223,6)" rx="2" ry="2" />
<text  x="1044.57" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (120 samples, 0.04%)</title><rect x="910.3" y="533" width="0.5" height="15.0" fill="rgb(212,92,31)" rx="2" ry="2" />
<text  x="913.35" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.NowFromContext (92 samples, 0.03%)</title><rect x="192.0" y="677" width="0.3" height="15.0" fill="rgb(213,149,5)" rx="2" ry="2" />
<text  x="195.02" y="687.5" ></text>
</g>
<g >
<title>runtime.startm (675 samples, 0.21%)</title><rect x="210.7" y="325" width="2.4" height="15.0" fill="rgb(209,226,47)" rx="2" ry="2" />
<text  x="213.70" y="335.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (56 samples, 0.02%)</title><rect x="65.8" y="501" width="0.2" height="15.0" fill="rgb(240,119,25)" rx="2" ry="2" />
<text  x="68.77" y="511.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,184 samples, 0.36%)</title><rect x="914.4" y="629" width="4.3" height="15.0" fill="rgb(231,20,43)" rx="2" ry="2" />
<text  x="917.45" y="639.5" ></text>
</g>
<g >
<title>runtime.stopm (1,013 samples, 0.31%)</title><rect x="1123.2" y="805" width="3.7" height="15.0" fill="rgb(234,119,2)" rx="2" ry="2" />
<text  x="1126.19" y="815.5" ></text>
</g>
<g >
<title>runtime.systemstack (155 samples, 0.05%)</title><rect x="1043.7" y="821" width="0.6" height="15.0" fill="rgb(252,111,5)" rx="2" ry="2" />
<text  x="1046.69" y="831.5" ></text>
</g>
<g >
<title>runtime.adjustpointers (56 samples, 0.02%)</title><rect x="1053.0" y="693" width="0.3" height="15.0" fill="rgb(213,72,16)" rx="2" ry="2" />
<text  x="1056.05" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (38 samples, 0.01%)</title><rect x="18.5" y="597" width="0.2" height="15.0" fill="rgb(251,43,35)" rx="2" ry="2" />
<text  x="21.53" y="607.5" ></text>
</g>
<g >
<title>runtime.runqgrab (47 samples, 0.01%)</title><rect x="1108.3" y="757" width="0.2" height="15.0" fill="rgb(219,108,23)" rx="2" ry="2" />
<text  x="1111.30" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).Readiness (35 samples, 0.01%)</title><rect x="170.0" y="741" width="0.1" height="15.0" fill="rgb(222,72,4)" rx="2" ry="2" />
<text  x="172.99" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MaxHeaderLength (103 samples, 0.03%)</title><rect x="1072.7" y="709" width="0.4" height="15.0" fill="rgb(227,201,31)" rx="2" ry="2" />
<text  x="1075.72" y="719.5" ></text>
</g>
<g >
<title>runtime.growslice (30 samples, 0.01%)</title><rect x="846.0" y="821" width="0.1" height="15.0" fill="rgb(230,44,20)" rx="2" ry="2" />
<text  x="848.97" y="831.5" ></text>
</g>
<g >
<title>runtime.mallocgc (40 samples, 0.01%)</title><rect x="1023.2" y="741" width="0.2" height="15.0" fill="rgb(217,61,44)" rx="2" ry="2" />
<text  x="1026.21" y="751.5" ></text>
</g>
<g >
<title>runtime.mallocgc (348 samples, 0.11%)</title><rect x="1041.6" y="757" width="1.3" height="15.0" fill="rgb(235,94,16)" rx="2" ry="2" />
<text  x="1044.64" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).contextValue (31 samples, 0.01%)</title><rect x="51.9" y="645" width="0.1" height="15.0" fill="rgb(208,8,0)" rx="2" ry="2" />
<text  x="54.87" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (38 samples, 0.01%)</title><rect x="117.9" y="517" width="0.2" height="15.0" fill="rgb(227,220,19)" rx="2" ry="2" />
<text  x="120.95" y="527.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (37 samples, 0.01%)</title><rect x="699.9" y="693" width="0.2" height="15.0" fill="rgb(229,136,45)" rx="2" ry="2" />
<text  x="702.95" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (59 samples, 0.02%)</title><rect x="452.3" y="645" width="0.2" height="15.0" fill="rgb(208,77,5)" rx="2" ry="2" />
<text  x="455.29" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (264 samples, 0.08%)</title><rect x="1122.2" y="661" width="0.9" height="15.0" fill="rgb(209,18,52)" rx="2" ry="2" />
<text  x="1125.19" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.buildTCPHdr (117 samples, 0.04%)</title><rect x="1093.4" y="757" width="0.4" height="15.0" fill="rgb(208,31,4)" rx="2" ry="2" />
<text  x="1096.42" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*context).Switch (131,203 samples, 40.21%)</title><rect x="221.6" y="837" width="474.4" height="15.0" fill="rgb(245,197,40)" rx="2" ry="2" />
<text  x="224.55" y="847.5" >gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*context).Switch</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (781 samples, 0.24%)</title><rect x="210.4" y="421" width="2.9" height="15.0" fill="rgb(242,44,50)" rx="2" ry="2" />
<text  x="213.43" y="431.5" ></text>
</g>
<g >
<title>runtime.adjustpointers (51 samples, 0.02%)</title><rect x="1102.8" y="693" width="0.2" height="15.0" fill="rgb(217,95,34)" rx="2" ry="2" />
<text  x="1105.78" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.writev (3,415 samples, 1.05%)</title><rect x="204.6" y="757" width="12.4" height="15.0" fill="rgb(229,161,14)" rx="2" ry="2" />
<text  x="207.60" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyOutBytes (551 samples, 0.17%)</title><rect x="61.2" y="741" width="2.0" height="15.0" fill="rgb(246,85,4)" rx="2" ry="2" />
<text  x="64.17" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (88 samples, 0.03%)</title><rect x="1051.5" y="741" width="0.3" height="15.0" fill="rgb(245,55,45)" rx="2" ry="2" />
<text  x="1054.52" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).getAddressOrCreateTempInner (505 samples, 0.15%)</title><rect x="1035.9" y="725" width="1.9" height="15.0" fill="rgb(222,78,19)" rx="2" ry="2" />
<text  x="1038.94" y="735.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).uncacheSpan (39 samples, 0.01%)</title><rect x="848.5" y="757" width="0.1" height="15.0" fill="rgb(240,107,8)" rx="2" ry="2" />
<text  x="851.49" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).resolvedFields (220 samples, 0.07%)</title><rect x="213.4" y="501" width="0.8" height="15.0" fill="rgb(243,76,29)" rx="2" ry="2" />
<text  x="216.41" y="511.5" ></text>
</g>
<g >
<title>runtime.goexit1 (59 samples, 0.02%)</title><rect x="32.7" y="885" width="0.2" height="15.0" fill="rgb(243,123,13)" rx="2" ry="2" />
<text  x="35.72" y="895.5" ></text>
</g>
<g >
<title>runtime.startm (374 samples, 0.11%)</title><rect x="1065.2" y="725" width="1.3" height="15.0" fill="rgb(207,2,19)" rx="2" ry="2" />
<text  x="1068.19" y="735.5" ></text>
</g>
<g >
<title>runtime.ready (158 samples, 0.05%)</title><rect x="1028.2" y="517" width="0.6" height="15.0" fill="rgb(238,20,25)" rx="2" ry="2" />
<text  x="1031.21" y="527.5" ></text>
</g>
<g >
<title>runtime.mallocgc (59 samples, 0.02%)</title><rect x="1043.4" y="805" width="0.2" height="15.0" fill="rgb(250,110,20)" rx="2" ry="2" />
<text  x="1046.42" y="815.5" ></text>
</g>
<g >
<title>runtime.adjusttimers (29 samples, 0.01%)</title><rect x="1059.8" y="725" width="0.1" height="15.0" fill="rgb(238,201,7)" rx="2" ry="2" />
<text  x="1062.81" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (31 samples, 0.01%)</title><rect x="1031.0" y="725" width="0.1" height="15.0" fill="rgb(220,188,36)" rx="2" ry="2" />
<text  x="1034.00" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,015 samples, 0.31%)</title><rect x="754.4" y="373" width="3.7" height="15.0" fill="rgb(217,193,51)" rx="2" ry="2" />
<text  x="757.40" y="383.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*EventPoll).RemoveEntry (331 samples, 0.10%)</title><rect x="67.2" y="709" width="1.2" height="15.0" fill="rgb(251,129,1)" rx="2" ry="2" />
<text  x="70.24" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (279 samples, 0.09%)</title><rect x="955.5" y="405" width="1.0" height="15.0" fill="rgb(217,87,22)" rx="2" ry="2" />
<text  x="958.54" y="415.5" ></text>
</g>
<g >
<title>runtime.copystack (1,159 samples, 0.36%)</title><rect x="1094.7" y="757" width="4.1" height="15.0" fill="rgb(230,210,19)" rx="2" ry="2" />
<text  x="1097.66" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (6,908 samples, 2.12%)</title><rect x="869.9" y="757" width="25.0" height="15.0" fill="rgb(213,196,48)" rx="2" ry="2" />
<text  x="872.91" y="767.5" >[..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,870 samples, 0.57%)</title><rect x="141.7" y="437" width="6.7" height="15.0" fill="rgb(210,119,21)" rx="2" ry="2" />
<text  x="144.65" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).destroy-fm (214 samples, 0.07%)</title><rect x="63.9" y="677" width="0.8" height="15.0" fill="rgb(214,102,20)" rx="2" ry="2" />
<text  x="66.93" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).NewFDs (204 samples, 0.06%)</title><rect x="46.3" y="709" width="0.8" height="15.0" fill="rgb(226,205,53)" rx="2" ry="2" />
<text  x="49.32" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*waiterList).Remove (54 samples, 0.02%)</title><rect x="170.8" y="741" width="0.2" height="15.0" fill="rgb(206,164,44)" rx="2" ry="2" />
<text  x="173.83" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportEndpoints).findEndpointLocked (28 samples, 0.01%)</title><rect x="775.1" y="693" width="0.1" height="15.0" fill="rgb(219,105,52)" rx="2" ry="2" />
<text  x="778.15" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*endpoint).dispatchLoop (52,179 samples, 15.99%)</title><rect x="706.9" y="853" width="188.7" height="15.0" fill="rgb(251,220,52)" rx="2" ry="2" />
<text  x="709.86" y="863.5" >gvisor.dev/gvisor/pkg/tc..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).startAcceptedLoop (453 samples, 0.14%)</title><rect x="1004.5" y="853" width="1.6" height="15.0" fill="rgb(241,106,40)" rx="2" ry="2" />
<text  x="1007.45" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).Done (30 samples, 0.01%)</title><rect x="1006.4" y="837" width="0.1" height="15.0" fill="rgb(235,170,52)" rx="2" ry="2" />
<text  x="1009.36" y="847.5" ></text>
</g>
<g >
<title>runtime.gfget (28 samples, 0.01%)</title><rect x="1048.2" y="661" width="0.1" height="15.0" fill="rgb(244,157,26)" rx="2" ry="2" />
<text  x="1051.18" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (41 samples, 0.01%)</title><rect x="1109.2" y="693" width="0.1" height="15.0" fill="rgb(220,144,38)" rx="2" ry="2" />
<text  x="1112.15" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/abi/linux.CopyEpollEventSliceOut (52 samples, 0.02%)</title><rect x="42.5" y="773" width="0.2" height="15.0" fill="rgb(254,8,51)" rx="2" ry="2" />
<text  x="45.52" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (44 samples, 0.01%)</title><rect x="170.4" y="725" width="0.2" height="15.0" fill="rgb(243,70,4)" rx="2" ry="2" />
<text  x="173.43" y="735.5" ></text>
</g>
<g >
<title>runtime.(*mheap).allocSpan (139 samples, 0.04%)</title><rect x="766.5" y="485" width="0.5" height="15.0" fill="rgb(213,69,28)" rx="2" ry="2" />
<text  x="769.55" y="495.5" ></text>
</g>
<g >
<title>runtime.wakep (695 samples, 0.21%)</title><rect x="210.6" y="341" width="2.5" height="15.0" fill="rgb(245,52,17)" rx="2" ry="2" />
<text  x="213.63" y="351.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.ExtractErrno (336 samples, 0.10%)</title><rect x="217.6" y="789" width="1.2" height="15.0" fill="rgb(248,131,5)" rx="2" ry="2" />
<text  x="220.57" y="799.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (40 samples, 0.01%)</title><rect x="1023.6" y="773" width="0.1" height="15.0" fill="rgb(216,147,2)" rx="2" ry="2" />
<text  x="1026.60" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (108 samples, 0.03%)</title><rect x="1083.1" y="453" width="0.4" height="15.0" fill="rgb(213,97,0)" rx="2" ry="2" />
<text  x="1086.07" y="463.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (216 samples, 0.07%)</title><rect x="1186.8" y="709" width="0.8" height="15.0" fill="rgb(242,64,3)" rx="2" ry="2" />
<text  x="1189.84" y="719.5" ></text>
</g>
<g >
<title>runtime.mallocgc (150 samples, 0.05%)</title><rect x="177.9" y="693" width="0.5" height="15.0" fill="rgb(251,33,15)" rx="2" ry="2" />
<text  x="180.87" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,459 samples, 0.75%)</title><rect x="437.8" y="629" width="8.9" height="15.0" fill="rgb(233,116,45)" rx="2" ry="2" />
<text  x="440.76" y="639.5" ></text>
</g>
<g >
<title>runtime.gopark (32 samples, 0.01%)</title><rect x="1066.9" y="821" width="0.1" height="15.0" fill="rgb(253,19,30)" rx="2" ry="2" />
<text  x="1069.92" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.Copy (39 samples, 0.01%)</title><rect x="75.1" y="661" width="0.1" height="15.0" fill="rgb(218,195,23)" rx="2" ry="2" />
<text  x="78.09" y="671.5" ></text>
</g>
<g >
<title>runtime.newstack (1,548 samples, 0.47%)</title><rect x="1013.6" y="773" width="5.6" height="15.0" fill="rgb(220,75,28)" rx="2" ry="2" />
<text  x="1016.56" y="783.5" ></text>
</g>
<g >
<title>runtime.notetsleep (149 samples, 0.05%)</title><rect x="12.9" y="757" width="0.5" height="15.0" fill="rgb(232,107,43)" rx="2" ry="2" />
<text  x="15.86" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (142 samples, 0.04%)</title><rect x="1189.2" y="597" width="0.5" height="15.0" fill="rgb(245,48,12)" rx="2" ry="2" />
<text  x="1192.19" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="29.2" y="821" width="0.2" height="15.0" fill="rgb(252,221,18)" rx="2" ry="2" />
<text  x="32.21" y="831.5" ></text>
</g>
<g >
<title>ext4_dirty_inode (226 samples, 0.07%)</title><rect x="200.1" y="261" width="0.8" height="15.0" fill="rgb(225,115,5)" rx="2" ry="2" />
<text  x="203.07" y="271.5" ></text>
</g>
<g >
<title>runtime.runqsteal (733 samples, 0.22%)</title><rect x="1120.5" y="805" width="2.7" height="15.0" fill="rgb(215,19,46)" rx="2" ry="2" />
<text  x="1123.54" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (212 samples, 0.06%)</title><rect x="1028.1" y="565" width="0.7" height="15.0" fill="rgb(218,227,3)" rx="2" ry="2" />
<text  x="1031.07" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (30 samples, 0.01%)</title><rect x="1090.3" y="757" width="0.1" height="15.0" fill="rgb(206,83,54)" rx="2" ry="2" />
<text  x="1093.33" y="767.5" ></text>
</g>
<g >
<title>runtime.stopm (181 samples, 0.06%)</title><rect x="22.6" y="805" width="0.7" height="15.0" fill="rgb(213,59,7)" rx="2" ry="2" />
<text  x="25.62" y="815.5" ></text>
</g>
<g >
<title>runtime.wirep (40 samples, 0.01%)</title><rect x="128.2" y="597" width="0.1" height="15.0" fill="rgb(228,214,29)" rx="2" ry="2" />
<text  x="131.20" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportEndpoints).iterEndpointsLocked (89 samples, 0.03%)</title><rect x="1067.5" y="789" width="0.3" height="15.0" fill="rgb(215,57,49)" rx="2" ry="2" />
<text  x="1070.47" y="799.5" ></text>
</g>
<g >
<title>runtime.mallocgc (32 samples, 0.01%)</title><rect x="1049.8" y="709" width="0.1" height="15.0" fill="rgb(236,215,6)" rx="2" ry="2" />
<text  x="1052.80" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/lock.(*Locks).UnlockRegion (51 samples, 0.02%)</title><rect x="69.2" y="725" width="0.2" height="15.0" fill="rgb(223,60,25)" rx="2" ry="2" />
<text  x="72.20" y="735.5" ></text>
</g>
<g >
<title>fdb_find_rcu (172 samples, 0.05%)</title><rect x="966.6" y="421" width="0.6" height="15.0" fill="rgb(215,100,40)" rx="2" ry="2" />
<text  x="969.60" y="431.5" ></text>
</g>
<g >
<title>runtime.schedule (1,350 samples, 0.41%)</title><rect x="1185.1" y="837" width="4.9" height="15.0" fill="rgb(243,174,42)" rx="2" ry="2" />
<text  x="1188.12" y="847.5" ></text>
</g>
<g >
<title>runtime.(*lfstack).pop (217 samples, 0.07%)</title><rect x="1142.9" y="693" width="0.8" height="15.0" fill="rgb(236,150,19)" rx="2" ry="2" />
<text  x="1145.87" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,143 samples, 0.96%)</title><rect x="810.8" y="709" width="11.4" height="15.0" fill="rgb(219,36,38)" rx="2" ry="2" />
<text  x="813.79" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).cleanupLocked (1,535 samples, 0.47%)</title><rect x="1099.9" y="837" width="5.5" height="15.0" fill="rgb(251,226,36)" rx="2" ry="2" />
<text  x="1102.86" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (48 samples, 0.01%)</title><rect x="701.8" y="645" width="0.2" height="15.0" fill="rgb(254,97,30)" rx="2" ry="2" />
<text  x="704.82" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut.func1 (72 samples, 0.02%)</title><rect x="73.6" y="709" width="0.3" height="15.0" fill="rgb(222,129,40)" rx="2" ry="2" />
<text  x="76.60" y="719.5" ></text>
</g>
<g >
<title>runtime.newobject (67 samples, 0.02%)</title><rect x="1081.3" y="613" width="0.3" height="15.0" fill="rgb(235,149,33)" rx="2" ry="2" />
<text  x="1084.31" y="623.5" ></text>
</g>
<g >
<title>runtime.mallocgc (47 samples, 0.01%)</title><rect x="1075.1" y="597" width="0.1" height="15.0" fill="rgb(220,115,5)" rx="2" ry="2" />
<text  x="1078.06" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).maxReceiveBufferSize (28 samples, 0.01%)</title><rect x="1038.9" y="757" width="0.1" height="15.0" fill="rgb(214,77,37)" rx="2" ry="2" />
<text  x="1041.88" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).Value (33 samples, 0.01%)</title><rect x="51.9" y="661" width="0.1" height="15.0" fill="rgb(242,78,37)" rx="2" ry="2" />
<text  x="54.86" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isResolutionRequiredRLocked (41 samples, 0.01%)</title><rect x="213.5" y="485" width="0.1" height="15.0" fill="rgb(254,22,20)" rx="2" ry="2" />
<text  x="216.48" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendData (2,359 samples, 0.72%)</title><rect x="207.8" y="693" width="8.6" height="15.0" fill="rgb(209,212,30)" rx="2" ry="2" />
<text  x="210.83" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="1173.1" y="773" width="0.1" height="15.0" fill="rgb(227,30,22)" rx="2" ry="2" />
<text  x="1176.09" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (63 samples, 0.02%)</title><rect x="845.7" y="805" width="0.3" height="15.0" fill="rgb(207,139,31)" rx="2" ry="2" />
<text  x="848.74" y="815.5" ></text>
</g>
<g >
<title>runtime.wakep (380 samples, 0.12%)</title><rect x="1065.2" y="741" width="1.3" height="15.0" fill="rgb(225,132,42)" rx="2" ry="2" />
<text  x="1068.17" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (44 samples, 0.01%)</title><rect x="905.4" y="709" width="0.1" height="15.0" fill="rgb(245,12,24)" rx="2" ry="2" />
<text  x="908.37" y="719.5" ></text>
</g>
<g >
<title>runtime.futex (124 samples, 0.04%)</title><rect x="157.4" y="597" width="0.4" height="15.0" fill="rgb(244,44,6)" rx="2" ry="2" />
<text  x="160.37" y="607.5" ></text>
</g>
<g >
<title>runtime.mallocgc (51 samples, 0.02%)</title><rect x="112.3" y="581" width="0.2" height="15.0" fill="rgb(212,82,17)" rx="2" ry="2" />
<text  x="115.27" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).WritePacket (636 samples, 0.19%)</title><rect x="1090.8" y="741" width="2.3" height="15.0" fill="rgb(222,136,44)" rx="2" ry="2" />
<text  x="1093.75" y="751.5" ></text>
</g>
<g >
<title>runtime/internal/atomic.Cas64 (45 samples, 0.01%)</title><rect x="758.3" y="597" width="0.2" height="15.0" fill="rgb(224,113,23)" rx="2" ry="2" />
<text  x="761.29" y="607.5" ></text>
</g>
<g >
<title>runtime.wakep (367 samples, 0.11%)</title><rect x="1127.2" y="805" width="1.4" height="15.0" fill="rgb(241,103,29)" rx="2" ry="2" />
<text  x="1130.24" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).decAddressRefLocked (103 samples, 0.03%)</title><rect x="776.6" y="677" width="0.4" height="15.0" fill="rgb(239,102,17)" rx="2" ry="2" />
<text  x="779.64" y="687.5" ></text>
</g>
<g >
<title>runtime.newobject (54 samples, 0.02%)</title><rect x="53.1" y="709" width="0.2" height="15.0" fill="rgb(242,171,11)" rx="2" ry="2" />
<text  x="56.10" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (50 samples, 0.02%)</title><rect x="52.0" y="661" width="0.2" height="15.0" fill="rgb(229,162,3)" rx="2" ry="2" />
<text  x="54.98" y="671.5" ></text>
</g>
<g >
<title>br_dev_queue_push_xmit (198 samples, 0.06%)</title><rect x="990.1" y="117" width="0.7" height="15.0" fill="rgb(241,214,17)" rx="2" ry="2" />
<text  x="993.08" y="127.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (264 samples, 0.08%)</title><rect x="14.9" y="645" width="1.0" height="15.0" fill="rgb(247,190,3)" rx="2" ry="2" />
<text  x="17.91" y="655.5" ></text>
</g>
<g >
<title>runtime.goschedImpl (244 samples, 0.07%)</title><rect x="1129.0" y="837" width="0.9" height="15.0" fill="rgb(253,150,10)" rx="2" ry="2" />
<text  x="1131.97" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*subprocess).switchToApp (114,574 samples, 35.11%)</title><rect x="223.9" y="821" width="414.4" height="15.0" fill="rgb(206,62,13)" rx="2" ry="2" />
<text  x="226.95" y="831.5" >gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*subproces..</text>
</g>
<g >
<title>runtime.markroot (57 samples, 0.02%)</title><rect x="1132.6" y="821" width="0.2" height="15.0" fill="rgb(232,193,9)" rx="2" ry="2" />
<text  x="1135.56" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).SyscallArgs (95 samples, 0.03%)</title><rect x="39.0" y="821" width="0.3" height="15.0" fill="rgb(245,196,30)" rx="2" ry="2" />
<text  x="41.95" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (58 samples, 0.02%)</title><rect x="1005.9" y="725" width="0.2" height="15.0" fill="rgb(206,170,30)" rx="2" ry="2" />
<text  x="1008.88" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*inodeOperations).UnstableAttr (47 samples, 0.01%)</title><rect x="191.3" y="661" width="0.1" height="15.0" fill="rgb(251,26,42)" rx="2" ry="2" />
<text  x="194.26" y="671.5" ></text>
</g>
<g >
<title>runtime.selparkcommit (126 samples, 0.04%)</title><rect x="158.0" y="661" width="0.5" height="15.0" fill="rgb(207,17,8)" rx="2" ry="2" />
<text  x="161.00" y="671.5" ></text>
</g>
<g >
<title>runtime.siftdownTimer (73 samples, 0.02%)</title><rect x="1046.2" y="725" width="0.3" height="15.0" fill="rgb(221,1,18)" rx="2" ry="2" />
<text  x="1049.20" y="735.5" ></text>
</g>
<g >
<title>runtime.scanobject (43 samples, 0.01%)</title><rect x="929.1" y="709" width="0.2" height="15.0" fill="rgb(231,143,23)" rx="2" ry="2" />
<text  x="932.10" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverNetworkPacket (21,623 samples, 6.63%)</title><rect x="718.4" y="805" width="78.2" height="15.0" fill="rgb(249,126,40)" rx="2" ry="2" />
<text  x="721.45" y="815.5" >gvisor.de..</text>
</g>
<g >
<title>runtime.park_m (278 samples, 0.09%)</title><rect x="1006.7" y="789" width="1.0" height="15.0" fill="rgb(205,224,32)" rx="2" ry="2" />
<text  x="1009.69" y="799.5" ></text>
</g>
<g >
<title>runtime.schedule (190 samples, 0.06%)</title><rect x="1129.1" y="821" width="0.7" height="15.0" fill="rgb(253,197,6)" rx="2" ry="2" />
<text  x="1132.13" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (201 samples, 0.06%)</title><rect x="15.1" y="613" width="0.8" height="15.0" fill="rgb(206,198,26)" rx="2" ry="2" />
<text  x="18.14" y="623.5" ></text>
</g>
<g >
<title>runtime.notewakeup (124 samples, 0.04%)</title><rect x="18.2" y="757" width="0.5" height="15.0" fill="rgb(224,175,15)" rx="2" ry="2" />
<text  x="21.22" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress (1,920 samples, 0.59%)</title><rect x="732.2" y="709" width="7.0" height="15.0" fill="rgb(236,148,53)" rx="2" ry="2" />
<text  x="735.21" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (31 samples, 0.01%)</title><rect x="704.9" y="773" width="0.1" height="15.0" fill="rgb(248,50,40)" rx="2" ry="2" />
<text  x="707.93" y="783.5" ></text>
</g>
<g >
<title>runtime.park_m (84 samples, 0.03%)</title><rect x="1111.3" y="837" width="0.3" height="15.0" fill="rgb(210,115,49)" rx="2" ry="2" />
<text  x="1114.25" y="847.5" ></text>
</g>
<g >
<title>runtime.chanrecv (165 samples, 0.05%)</title><rect x="451.6" y="757" width="0.6" height="15.0" fill="rgb(251,162,25)" rx="2" ry="2" />
<text  x="454.64" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*runApp).execute (182,944 samples, 56.07%)</title><rect x="34.8" y="853" width="661.6" height="15.0" fill="rgb(246,64,53)" rx="2" ry="2" />
<text  x="37.78" y="863.5" >gvisor.dev/gvisor/pkg/sentry/kernel.(*runApp).execute</text>
</g>
<g >
<title>[[kernel.kallsyms]] (139 samples, 0.04%)</title><rect x="1119.2" y="693" width="0.5" height="15.0" fill="rgb(244,217,11)" rx="2" ry="2" />
<text  x="1122.22" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).internalMappingsLocked (78 samples, 0.02%)</title><rect x="76.1" y="693" width="0.3" height="15.0" fill="rgb(214,195,4)" rx="2" ry="2" />
<text  x="79.14" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).clone (343 samples, 0.11%)</title><rect x="703.8" y="853" width="1.3" height="15.0" fill="rgb(244,141,52)" rx="2" ry="2" />
<text  x="706.85" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="199.0" y="229" width="0.2" height="15.0" fill="rgb(243,224,19)" rx="2" ry="2" />
<text  x="202.04" y="239.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (38 samples, 0.01%)</title><rect x="848.0" y="677" width="0.1" height="15.0" fill="rgb(212,101,11)" rx="2" ry="2" />
<text  x="850.95" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacket (113 samples, 0.03%)</title><rect x="181.0" y="469" width="0.4" height="15.0" fill="rgb(227,120,38)" rx="2" ry="2" />
<text  x="184.04" y="479.5" ></text>
</g>
<g >
<title>runtime.makemap_small (33 samples, 0.01%)</title><rect x="49.8" y="661" width="0.2" height="15.0" fill="rgb(235,116,31)" rx="2" ry="2" />
<text  x="52.84" y="671.5" ></text>
</g>
<g >
<title>runtime.assertI2I2 (39 samples, 0.01%)</title><rect x="1038.3" y="725" width="0.2" height="15.0" fill="rgb(236,159,52)" rx="2" ry="2" />
<text  x="1041.32" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (53 samples, 0.02%)</title><rect x="1010.9" y="709" width="0.2" height="15.0" fill="rgb(222,194,6)" rx="2" ry="2" />
<text  x="1013.92" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut.func1 (224 samples, 0.07%)</title><rect x="61.8" y="693" width="0.8" height="15.0" fill="rgb(240,198,15)" rx="2" ry="2" />
<text  x="64.77" y="703.5" ></text>
</g>
<g >
<title>ctnetlink_conntrack_event (182 samples, 0.06%)</title><rect x="993.1" y="277" width="0.6" height="15.0" fill="rgb(244,68,12)" rx="2" ry="2" />
<text  x="996.05" y="287.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (57 samples, 0.02%)</title><rect x="23.1" y="629" width="0.2" height="15.0" fill="rgb(227,34,41)" rx="2" ry="2" />
<text  x="26.07" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (8,667 samples, 2.66%)</title><rect x="664.3" y="725" width="31.3" height="15.0" fill="rgb(215,161,22)" rx="2" ry="2" />
<text  x="667.26" y="735.5" >[[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).SwapAnd (111 samples, 0.03%)</title><rect x="1051.5" y="757" width="0.4" height="15.0" fill="rgb(209,118,22)" rx="2" ry="2" />
<text  x="1054.45" y="767.5" ></text>
</g>
<g >
<title>runtime.morestack (1,033 samples, 0.32%)</title><rect x="1051.9" y="773" width="3.8" height="15.0" fill="rgb(212,83,24)" rx="2" ry="2" />
<text  x="1054.93" y="783.5" ></text>
</g>
<g >
<title>runtime.startm (66 samples, 0.02%)</title><rect x="28.9" y="837" width="0.3" height="15.0" fill="rgb(238,156,49)" rx="2" ry="2" />
<text  x="31.94" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn (110 samples, 0.03%)</title><rect x="56.7" y="677" width="0.4" height="15.0" fill="rgb(231,53,44)" rx="2" ry="2" />
<text  x="59.70" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,400 samples, 0.43%)</title><rect x="951.5" y="517" width="5.0" height="15.0" fill="rgb(249,83,2)" rx="2" ry="2" />
<text  x="954.48" y="527.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).startUpdater.func1 (44 samples, 0.01%)</title><rect x="696.8" y="869" width="0.2" height="15.0" fill="rgb(226,70,30)" rx="2" ry="2" />
<text  x="699.81" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (56 samples, 0.02%)</title><rect x="701.8" y="693" width="0.2" height="15.0" fill="rgb(213,196,13)" rx="2" ry="2" />
<text  x="704.80" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (258 samples, 0.08%)</title><rect x="114.9" y="581" width="1.0" height="15.0" fill="rgb(228,52,50)" rx="2" ry="2" />
<text  x="117.95" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (68 samples, 0.02%)</title><rect x="1189.7" y="757" width="0.3" height="15.0" fill="rgb(233,113,47)" rx="2" ry="2" />
<text  x="1192.75" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (807 samples, 0.25%)</title><rect x="953.6" y="469" width="2.9" height="15.0" fill="rgb(240,173,44)" rx="2" ry="2" />
<text  x="956.63" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (28 samples, 0.01%)</title><rect x="57.5" y="693" width="0.1" height="15.0" fill="rgb(241,157,22)" rx="2" ry="2" />
<text  x="60.47" y="703.5" ></text>
</g>
<g >
<title>runtime.(*itabTableType).find (41 samples, 0.01%)</title><rect x="179.0" y="725" width="0.1" height="15.0" fill="rgb(212,165,5)" rx="2" ry="2" />
<text  x="181.98" y="735.5" ></text>
</g>
<g >
<title>runtime.futexsleep (56 samples, 0.02%)</title><rect x="128.4" y="613" width="0.2" height="15.0" fill="rgb(242,174,51)" rx="2" ry="2" />
<text  x="131.43" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (53 samples, 0.02%)</title><rect x="738.7" y="661" width="0.2" height="15.0" fill="rgb(241,141,39)" rx="2" ry="2" />
<text  x="741.69" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).resetKickerLocked (51 samples, 0.02%)</title><rect x="162.5" y="709" width="0.2" height="15.0" fill="rgb(207,198,3)" rx="2" ry="2" />
<text  x="165.48" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).newHandshake (218 samples, 0.07%)</title><rect x="1024.4" y="789" width="0.8" height="15.0" fill="rgb(219,222,7)" rx="2" ry="2" />
<text  x="1027.41" y="799.5" ></text>
</g>
<g >
<title>runtime.newobject (68 samples, 0.02%)</title><rect x="1050.2" y="773" width="0.2" height="15.0" fill="rgb(222,166,51)" rx="2" ry="2" />
<text  x="1053.17" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn (127 samples, 0.04%)</title><rect x="69.6" y="725" width="0.5" height="15.0" fill="rgb(212,18,49)" rx="2" ry="2" />
<text  x="72.64" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*overlayFileOperations).Write (3,088 samples, 0.95%)</title><rect x="191.8" y="725" width="11.1" height="15.0" fill="rgb(238,82,20)" rx="2" ry="2" />
<text  x="194.77" y="735.5" ></text>
</g>
<g >
<title>[[vdso]] (615 samples, 0.19%)</title><rect x="26.7" y="837" width="2.2" height="15.0" fill="rgb(219,188,17)" rx="2" ry="2" />
<text  x="29.71" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,661 samples, 0.82%)</title><rect x="812.5" y="693" width="9.7" height="15.0" fill="rgb(215,190,32)" rx="2" ry="2" />
<text  x="815.54" y="703.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (61 samples, 0.02%)</title><rect x="178.2" y="677" width="0.2" height="15.0" fill="rgb(214,151,47)" rx="2" ry="2" />
<text  x="181.16" y="687.5" ></text>
</g>
<g >
<title>runtime.resetspinning (215 samples, 0.07%)</title><rect x="702.1" y="789" width="0.8" height="15.0" fill="rgb(225,6,25)" rx="2" ry="2" />
<text  x="705.09" y="799.5" ></text>
</g>
<g >
<title>runtime.systemstack (531 samples, 0.16%)</title><rect x="843.1" y="789" width="1.9" height="15.0" fill="rgb(252,58,14)" rx="2" ry="2" />
<text  x="846.08" y="799.5" ></text>
</g>
<g >
<title>br_nf_pre_routing (10,799 samples, 3.31%)</title><rect x="957.6" y="501" width="39.1" height="15.0" fill="rgb(231,92,27)" rx="2" ry="2" />
<text  x="960.64" y="511.5" >br_..</text>
</g>
<g >
<title>runtime.futex (124 samples, 0.04%)</title><rect x="18.2" y="741" width="0.5" height="15.0" fill="rgb(240,72,46)" rx="2" ry="2" />
<text  x="21.22" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (166 samples, 0.05%)</title><rect x="176.3" y="629" width="0.6" height="15.0" fill="rgb(214,87,27)" rx="2" ry="2" />
<text  x="179.32" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (97 samples, 0.03%)</title><rect x="1049.4" y="533" width="0.3" height="15.0" fill="rgb(231,191,42)" rx="2" ry="2" />
<text  x="1052.39" y="543.5" ></text>
</g>
<g >
<title>runtime.gcDrainN (28 samples, 0.01%)</title><rect x="1068.3" y="741" width="0.1" height="15.0" fill="rgb(228,31,22)" rx="2" ry="2" />
<text  x="1071.34" y="751.5" ></text>
</g>
<g >
<title>runtime.mallocgc (36 samples, 0.01%)</title><rect x="1079.8" y="629" width="0.1" height="15.0" fill="rgb(206,65,50)" rx="2" ry="2" />
<text  x="1082.75" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (67 samples, 0.02%)</title><rect x="1074.0" y="533" width="0.3" height="15.0" fill="rgb(245,163,9)" rx="2" ry="2" />
<text  x="1077.02" y="543.5" ></text>
</g>
<g >
<title>ipv4_conntrack_defrag (85 samples, 0.03%)</title><rect x="959.4" y="469" width="0.3" height="15.0" fill="rgb(252,6,40)" rx="2" ry="2" />
<text  x="962.36" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).setRegs (32 samples, 0.01%)</title><rect x="229.4" y="805" width="0.1" height="15.0" fill="rgb(226,136,1)" rx="2" ry="2" />
<text  x="232.39" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*IPTables).Check (30 samples, 0.01%)</title><rect x="781.8" y="741" width="0.1" height="15.0" fill="rgb(233,102,19)" rx="2" ry="2" />
<text  x="784.84" y="751.5" ></text>
</g>
<g >
<title>runtime.newobject (54 samples, 0.02%)</title><rect x="1050.0" y="741" width="0.2" height="15.0" fill="rgb(212,77,51)" rx="2" ry="2" />
<text  x="1052.96" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (112 samples, 0.03%)</title><rect x="148.9" y="613" width="0.5" height="15.0" fill="rgb(236,19,17)" rx="2" ry="2" />
<text  x="151.94" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,218 samples, 0.99%)</title><rect x="829.6" y="693" width="11.6" height="15.0" fill="rgb(245,167,7)" rx="2" ry="2" />
<text  x="832.61" y="703.5" ></text>
</g>
<g >
<title>runtime.notewakeup (133 samples, 0.04%)</title><rect x="13.5" y="741" width="0.5" height="15.0" fill="rgb(248,94,12)" rx="2" ry="2" />
<text  x="16.50" y="751.5" ></text>
</g>
<g >
<title>runtime.adjustframe (507 samples, 0.16%)</title><rect x="1148.0" y="725" width="1.9" height="15.0" fill="rgb(251,138,34)" rx="2" ry="2" />
<text  x="1151.04" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (45 samples, 0.01%)</title><rect x="1076.2" y="693" width="0.2" height="15.0" fill="rgb(215,115,54)" rx="2" ry="2" />
<text  x="1079.19" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).GetKind (184 samples, 0.06%)</title><rect x="736.7" y="661" width="0.7" height="15.0" fill="rgb(210,46,6)" rx="2" ry="2" />
<text  x="739.74" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*inodeFileState).WriteFromBlocksAt (2,536 samples, 0.78%)</title><rect x="193.0" y="597" width="9.2" height="15.0" fill="rgb(244,117,14)" rx="2" ry="2" />
<text  x="196.03" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).isEndpointWritableLocked (40 samples, 0.01%)</title><rect x="207.1" y="677" width="0.2" height="15.0" fill="rgb(250,226,50)" rx="2" ry="2" />
<text  x="210.12" y="687.5" ></text>
</g>
<g >
<title>runtime/internal/atomic.Xchg64 (47 samples, 0.01%)</title><rect x="758.6" y="613" width="0.2" height="15.0" fill="rgb(235,74,51)" rx="2" ry="2" />
<text  x="761.62" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (70 samples, 0.02%)</title><rect x="639.4" y="821" width="0.2" height="15.0" fill="rgb(253,120,37)" rx="2" ry="2" />
<text  x="642.36" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (56 samples, 0.02%)</title><rect x="788.2" y="629" width="0.2" height="15.0" fill="rgb(252,129,15)" rx="2" ry="2" />
<text  x="791.16" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyInBytes (133 samples, 0.04%)</title><rect x="69.6" y="741" width="0.5" height="15.0" fill="rgb(246,74,33)" rx="2" ry="2" />
<text  x="72.62" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress (35 samples, 0.01%)</title><rect x="782.3" y="725" width="0.2" height="15.0" fill="rgb(252,217,13)" rx="2" ry="2" />
<text  x="785.34" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).Deprecated (73 samples, 0.02%)</title><rect x="725.9" y="677" width="0.2" height="15.0" fill="rgb(213,228,16)" rx="2" ry="2" />
<text  x="728.88" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleSynSegment.func1 (5,977 samples, 1.83%)</title><rect x="999.7" y="869" width="21.6" height="15.0" fill="rgb(240,84,10)" rx="2" ry="2" />
<text  x="1002.69" y="879.5" >g..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (47 samples, 0.01%)</title><rect x="181.1" y="421" width="0.2" height="15.0" fill="rgb(233,82,37)" rx="2" ry="2" />
<text  x="184.10" y="431.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (838 samples, 0.26%)</title><rect x="198.1" y="325" width="3.0" height="15.0" fill="rgb(253,204,45)" rx="2" ry="2" />
<text  x="201.06" y="335.5" ></text>
</g>
<g >
<title>runtime.getStackMap (357 samples, 0.11%)</title><rect x="1148.6" y="709" width="1.3" height="15.0" fill="rgb(242,204,25)" rx="2" ry="2" />
<text  x="1151.57" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (41 samples, 0.01%)</title><rect x="1154.4" y="741" width="0.1" height="15.0" fill="rgb(228,56,5)" rx="2" ry="2" />
<text  x="1157.39" y="751.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (30 samples, 0.01%)</title><rect x="184.7" y="661" width="0.1" height="15.0" fill="rgb(237,169,8)" rx="2" ry="2" />
<text  x="187.66" y="671.5" ></text>
</g>
<g >
<title>[unknown] (1,006 samples, 0.31%)</title><rect x="12.7" y="837" width="3.6" height="15.0" fill="rgb(245,213,38)" rx="2" ry="2" />
<text  x="15.70" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (48 samples, 0.01%)</title><rect x="1060.6" y="549" width="0.2" height="15.0" fill="rgb(235,186,29)" rx="2" ry="2" />
<text  x="1063.63" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).updateMaxPayloadSize (37 samples, 0.01%)</title><rect x="1010.5" y="757" width="0.1" height="15.0" fill="rgb(252,173,38)" rx="2" ry="2" />
<text  x="1013.50" y="767.5" ></text>
</g>
<g >
<title>runtime.startm (554 samples, 0.17%)</title><rect x="1187.7" y="789" width="2.0" height="15.0" fill="rgb(230,110,3)" rx="2" ry="2" />
<text  x="1190.72" y="799.5" ></text>
</g>
<g >
<title>runtime.step (64 samples, 0.02%)</title><rect x="1150.4" y="693" width="0.2" height="15.0" fill="rgb(252,171,32)" rx="2" ry="2" />
<text  x="1153.40" y="703.5" ></text>
</g>
<g >
<title>runtime.mapassign (62 samples, 0.02%)</title><rect x="72.2" y="725" width="0.3" height="15.0" fill="rgb(220,192,54)" rx="2" ry="2" />
<text  x="75.23" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,236 samples, 0.38%)</title><rect x="753.6" y="389" width="4.5" height="15.0" fill="rgb(241,48,16)" rx="2" ry="2" />
<text  x="756.60" y="399.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (77 samples, 0.02%)</title><rect x="1189.4" y="581" width="0.3" height="15.0" fill="rgb(225,166,3)" rx="2" ry="2" />
<text  x="1192.43" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (235 samples, 0.07%)</title><rect x="704.0" y="613" width="0.9" height="15.0" fill="rgb(218,84,54)" rx="2" ry="2" />
<text  x="707.03" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (75 samples, 0.02%)</title><rect x="446.4" y="565" width="0.2" height="15.0" fill="rgb(224,74,28)" rx="2" ry="2" />
<text  x="449.37" y="575.5" ></text>
</g>
<g >
<title>jbd2__journal_start (92 samples, 0.03%)</title><rect x="199.4" y="293" width="0.3" height="15.0" fill="rgb(248,87,21)" rx="2" ry="2" />
<text  x="202.41" y="303.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).DecRef (34 samples, 0.01%)</title><rect x="1100.8" y="805" width="0.1" height="15.0" fill="rgb(239,220,43)" rx="2" ry="2" />
<text  x="1103.81" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (40 samples, 0.01%)</title><rect x="1051.2" y="597" width="0.2" height="15.0" fill="rgb(224,98,38)" rx="2" ry="2" />
<text  x="1054.21" y="607.5" ></text>
</g>
<g >
<title>runtime.lock2 (345 samples, 0.11%)</title><rect x="159.5" y="677" width="1.3" height="15.0" fill="rgb(247,81,47)" rx="2" ry="2" />
<text  x="162.53" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/packetsocket.(*endpoint).WritePacket (181 samples, 0.06%)</title><rect x="1091.3" y="629" width="0.7" height="15.0" fill="rgb(207,48,26)" rx="2" ry="2" />
<text  x="1094.30" y="639.5" ></text>
</g>
<g >
<title>runtime.newobject (60 samples, 0.02%)</title><rect x="1008.8" y="757" width="0.2" height="15.0" fill="rgb(233,54,24)" rx="2" ry="2" />
<text  x="1011.78" y="767.5" ></text>
</g>
<g >
<title>runtime.resetspinning (112 samples, 0.03%)</title><rect x="23.3" y="821" width="0.4" height="15.0" fill="rgb(218,114,39)" rx="2" ry="2" />
<text  x="26.30" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="23.6" y="661" width="0.1" height="15.0" fill="rgb(211,149,29)" rx="2" ry="2" />
<text  x="26.55" y="671.5" ></text>
</g>
<g >
<title>runtime.runqempty (37 samples, 0.01%)</title><rect x="1128.6" y="821" width="0.1" height="15.0" fill="rgb(226,159,37)" rx="2" ry="2" />
<text  x="1131.57" y="831.5" ></text>
</g>
<g >
<title>runtime.stackalloc (266 samples, 0.08%)</title><rect x="1097.8" y="741" width="1.0" height="15.0" fill="rgb(221,128,43)" rx="2" ry="2" />
<text  x="1100.84" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,746 samples, 1.45%)</title><rect x="805.0" y="773" width="17.2" height="15.0" fill="rgb(234,174,33)" rx="2" ry="2" />
<text  x="808.00" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (132 samples, 0.04%)</title><rect x="904.4" y="709" width="0.5" height="15.0" fill="rgb(246,82,45)" rx="2" ry="2" />
<text  x="907.43" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut (921 samples, 0.28%)</title><rect x="73.4" y="725" width="3.3" height="15.0" fill="rgb(250,122,31)" rx="2" ry="2" />
<text  x="76.39" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).entry (77 samples, 0.02%)</title><rect x="1074.7" y="597" width="0.3" height="15.0" fill="rgb(206,131,4)" rx="2" ry="2" />
<text  x="1077.71" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (425 samples, 0.13%)</title><rect x="125.8" y="453" width="1.6" height="15.0" fill="rgb(243,39,39)" rx="2" ry="2" />
<text  x="128.83" y="463.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (195 samples, 0.06%)</title><rect x="1130.2" y="741" width="0.7" height="15.0" fill="rgb(221,111,49)" rx="2" ry="2" />
<text  x="1133.20" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (17,124 samples, 5.25%)</title><rect x="936.2" y="757" width="61.9" height="15.0" fill="rgb(219,4,50)" rx="2" ry="2" />
<text  x="939.21" y="767.5" >[[kern..</text>
</g>
<g >
<title>runtime.notewakeup (61 samples, 0.02%)</title><rect x="1051.1" y="645" width="0.3" height="15.0" fill="rgb(229,158,49)" rx="2" ry="2" />
<text  x="1054.13" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (322 samples, 0.10%)</title><rect x="126.2" y="437" width="1.2" height="15.0" fill="rgb(238,26,28)" rx="2" ry="2" />
<text  x="129.21" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).SetReturn (30 samples, 0.01%)</title><rect x="36.5" y="837" width="0.1" height="15.0" fill="rgb(254,138,10)" rx="2" ry="2" />
<text  x="39.48" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (92 samples, 0.03%)</title><rect x="18.3" y="709" width="0.4" height="15.0" fill="rgb(221,205,44)" rx="2" ry="2" />
<text  x="21.33" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).nextWaker (7,134 samples, 2.19%)</title><rect x="896.7" y="821" width="25.8" height="15.0" fill="rgb(240,84,52)" rx="2" ry="2" />
<text  x="899.74" y="831.5" >g..</text>
</g>
<g >
<title>runtime.copystack (1,283 samples, 0.39%)</title><rect x="1013.7" y="757" width="4.6" height="15.0" fill="rgb(236,178,13)" rx="2" ry="2" />
<text  x="1016.70" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*pollEntry).Callback (134 samples, 0.04%)</title><rect x="1071.4" y="741" width="0.5" height="15.0" fill="rgb(235,14,27)" rx="2" ry="2" />
<text  x="1074.42" y="751.5" ></text>
</g>
<g >
<title>runtime.newproc1 (202 samples, 0.06%)</title><rect x="1080.2" y="613" width="0.7" height="15.0" fill="rgb(231,54,51)" rx="2" ry="2" />
<text  x="1083.22" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).contextValue (42 samples, 0.01%)</title><rect x="190.8" y="693" width="0.1" height="15.0" fill="rgb(208,33,23)" rx="2" ry="2" />
<text  x="193.76" y="703.5" ></text>
</g>
<g >
<title>runtime.stopm (34 samples, 0.01%)</title><rect x="1129.5" y="789" width="0.2" height="15.0" fill="rgb(236,68,11)" rx="2" ry="2" />
<text  x="1132.55" y="799.5" ></text>
</g>
<g >
<title>runtime.adjustframe (544 samples, 0.17%)</title><rect x="1095.0" y="725" width="2.0" height="15.0" fill="rgb(238,197,35)" rx="2" ry="2" />
<text  x="1098.03" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).WritePacket (735 samples, 0.23%)</title><rect x="1073.2" y="709" width="2.7" height="15.0" fill="rgb(250,160,27)" rx="2" ry="2" />
<text  x="1076.20" y="719.5" ></text>
</g>
<g >
<title>runtime.scanobject (44 samples, 0.01%)</title><rect x="1106.6" y="757" width="0.2" height="15.0" fill="rgb(221,112,28)" rx="2" ry="2" />
<text  x="1109.64" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (46 samples, 0.01%)</title><rect x="736.2" y="661" width="0.2" height="15.0" fill="rgb(218,144,48)" rx="2" ry="2" />
<text  x="739.19" y="671.5" ></text>
</g>
<g >
<title>runtime.startm (142 samples, 0.04%)</title><rect x="13.5" y="757" width="0.5" height="15.0" fill="rgb(250,162,0)" rx="2" ry="2" />
<text  x="16.46" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).WritePacket (364 samples, 0.11%)</title><rect x="180.8" y="549" width="1.3" height="15.0" fill="rgb(221,215,30)" rx="2" ry="2" />
<text  x="183.82" y="559.5" ></text>
</g>
<g >
<title>setup_pre_routing (392 samples, 0.12%)</title><rect x="995.3" y="485" width="1.4" height="15.0" fill="rgb(226,143,38)" rx="2" ry="2" />
<text  x="998.28" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).shutdownLocked (1,031 samples, 0.32%)</title><rect x="179.7" y="725" width="3.8" height="15.0" fill="rgb(223,216,9)" rx="2" ry="2" />
<text  x="182.72" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (734 samples, 0.22%)</title><rect x="444.0" y="597" width="2.7" height="15.0" fill="rgb(236,122,9)" rx="2" ry="2" />
<text  x="447.00" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.New (1,650 samples, 0.51%)</title><rect x="47.3" y="725" width="6.0" height="15.0" fill="rgb(251,112,9)" rx="2" ry="2" />
<text  x="50.33" y="735.5" ></text>
</g>
<g >
<title>runtime.park_m (532 samples, 0.16%)</title><rect x="21.8" y="853" width="1.9" height="15.0" fill="rgb(237,188,14)" rx="2" ry="2" />
<text  x="24.81" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (230 samples, 0.07%)</title><rect x="1122.3" y="645" width="0.8" height="15.0" fill="rgb(237,146,3)" rx="2" ry="2" />
<text  x="1125.32" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="905.4" y="693" width="0.1" height="15.0" fill="rgb(227,68,27)" rx="2" ry="2" />
<text  x="908.39" y="703.5" ></text>
</g>
<g >
<title>runtime.futex (577 samples, 0.18%)</title><rect x="1062.7" y="709" width="2.1" height="15.0" fill="rgb(222,50,17)" rx="2" ry="2" />
<text  x="1065.71" y="719.5" ></text>
</g>
<g >
<title>runtime.write1 (112 samples, 0.03%)</title><rect x="148.9" y="629" width="0.5" height="15.0" fill="rgb(226,78,22)" rx="2" ry="2" />
<text  x="151.94" y="639.5" ></text>
</g>
<g >
<title>runtime.step (85 samples, 0.03%)</title><rect x="1054.0" y="645" width="0.3" height="15.0" fill="rgb(238,204,11)" rx="2" ry="2" />
<text  x="1056.97" y="655.5" ></text>
</g>
<g >
<title>start_this_handle (39 samples, 0.01%)</title><rect x="199.6" y="277" width="0.1" height="15.0" fill="rgb(206,172,49)" rx="2" ry="2" />
<text  x="202.60" y="287.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (91 samples, 0.03%)</title><rect x="1108.7" y="661" width="0.3" height="15.0" fill="rgb(218,162,42)" rx="2" ry="2" />
<text  x="1111.71" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.TcpipAfterFunc (1,461 samples, 0.45%)</title><rect x="1078.9" y="693" width="5.3" height="15.0" fill="rgb(251,21,22)" rx="2" ry="2" />
<text  x="1081.90" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).WritePacket (100 samples, 0.03%)</title><rect x="181.1" y="453" width="0.3" height="15.0" fill="rgb(211,206,51)" rx="2" ry="2" />
<text  x="184.07" y="463.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (215 samples, 0.07%)</title><rect x="74.8" y="677" width="0.7" height="15.0" fill="rgb(238,81,19)" rx="2" ry="2" />
<text  x="77.77" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (117 samples, 0.04%)</title><rect x="166.7" y="725" width="0.4" height="15.0" fill="rgb(217,143,18)" rx="2" ry="2" />
<text  x="169.71" y="735.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (141 samples, 0.04%)</title><rect x="711.1" y="741" width="0.5" height="15.0" fill="rgb(212,180,14)" rx="2" ry="2" />
<text  x="714.12" y="751.5" ></text>
</g>
<g >
<title>runtime.schedule (18,084 samples, 5.54%)</title><rect x="92.6" y="661" width="65.4" height="15.0" fill="rgb(222,48,32)" rx="2" ry="2" />
<text  x="95.60" y="671.5" >runtime..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*listenContext).createConnectingEndpoint (3,014 samples, 0.92%)</title><rect x="1032.2" y="805" width="10.9" height="15.0" fill="rgb(206,146,25)" rx="2" ry="2" />
<text  x="1035.18" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/binary.Marshal (29 samples, 0.01%)</title><rect x="923.2" y="821" width="0.1" height="15.0" fill="rgb(229,62,48)" rx="2" ry="2" />
<text  x="926.19" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (265 samples, 0.08%)</title><rect x="1063.8" y="581" width="1.0" height="15.0" fill="rgb(245,69,50)" rx="2" ry="2" />
<text  x="1066.84" y="591.5" ></text>
</g>
<g >
<title>runtime.epollwait (52 samples, 0.02%)</title><rect x="699.9" y="773" width="0.2" height="15.0" fill="rgb(254,228,37)" rx="2" ry="2" />
<text  x="702.89" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).resolvedFields (189 samples, 0.06%)</title><rect x="1092.1" y="677" width="0.6" height="15.0" fill="rgb(244,194,11)" rx="2" ry="2" />
<text  x="1095.05" y="687.5" ></text>
</g>
<g >
<title>runtime.scanobject (52 samples, 0.02%)</title><rect x="1020.2" y="741" width="0.2" height="15.0" fill="rgb(206,167,42)" rx="2" ry="2" />
<text  x="1023.18" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).nextWaker (286 samples, 0.09%)</title><rect x="1045.7" y="837" width="1.0" height="15.0" fill="rgb(231,211,28)" rx="2" ry="2" />
<text  x="1048.66" y="847.5" ></text>
</g>
<g >
<title>runtime.clone (47 samples, 0.01%)</title><rect x="16.8" y="853" width="0.1" height="15.0" fill="rgb(225,0,6)" rx="2" ry="2" />
<text  x="19.75" y="863.5" ></text>
</g>
<g >
<title>syscall.Wait4 (66 samples, 0.02%)</title><rect x="638.0" y="805" width="0.3" height="15.0" fill="rgb(213,115,40)" rx="2" ry="2" />
<text  x="641.02" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (252 samples, 0.08%)</title><rect x="59.8" y="709" width="1.0" height="15.0" fill="rgb(230,121,27)" rx="2" ry="2" />
<text  x="62.84" y="719.5" ></text>
</g>
<g >
<title>runtime.mcall (281 samples, 0.09%)</title><rect x="1006.7" y="805" width="1.0" height="15.0" fill="rgb(208,158,2)" rx="2" ry="2" />
<text  x="1009.67" y="815.5" ></text>
</g>
<g >
<title>runtime.deltimer (33 samples, 0.01%)</title><rect x="1085.5" y="661" width="0.1" height="15.0" fill="rgb(250,173,14)" rx="2" ry="2" />
<text  x="1088.52" y="671.5" ></text>
</g>
<g >
<title>runtime.resetspinning (1,851 samples, 0.57%)</title><rect x="149.7" y="645" width="6.7" height="15.0" fill="rgb(252,72,35)" rx="2" ry="2" />
<text  x="152.67" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (48 samples, 0.01%)</title><rect x="1153.3" y="661" width="0.2" height="15.0" fill="rgb(241,204,1)" rx="2" ry="2" />
<text  x="1156.34" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.NewWeakRef (279 samples, 0.09%)</title><rect x="70.5" y="725" width="1.0" height="15.0" fill="rgb(209,20,14)" rx="2" ry="2" />
<text  x="73.47" y="735.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc1 (49 samples, 0.02%)</title><rect x="1106.6" y="789" width="0.2" height="15.0" fill="rgb(216,140,15)" rx="2" ry="2" />
<text  x="1109.62" y="799.5" ></text>
</g>
<g >
<title>runtime.mapdelete (69 samples, 0.02%)</title><rect x="68.1" y="693" width="0.2" height="15.0" fill="rgb(235,41,54)" rx="2" ry="2" />
<text  x="71.06" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (251 samples, 0.08%)</title><rect x="115.0" y="565" width="0.9" height="15.0" fill="rgb(211,31,24)" rx="2" ry="2" />
<text  x="117.97" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (449 samples, 0.14%)</title><rect x="14.2" y="741" width="1.7" height="15.0" fill="rgb(231,33,51)" rx="2" ry="2" />
<text  x="17.24" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (138 samples, 0.04%)</title><rect x="1049.2" y="565" width="0.5" height="15.0" fill="rgb(210,139,0)" rx="2" ry="2" />
<text  x="1052.24" y="575.5" ></text>
</g>
<g >
<title>runtime.mallocgc (260 samples, 0.08%)</title><rect x="763.2" y="581" width="0.9" height="15.0" fill="rgb(208,49,35)" rx="2" ry="2" />
<text  x="766.15" y="591.5" ></text>
</g>
<g >
<title>runtime.mcall (225 samples, 0.07%)</title><rect x="1045.8" y="821" width="0.8" height="15.0" fill="rgb(242,77,31)" rx="2" ry="2" />
<text  x="1048.77" y="831.5" ></text>
</g>
<g >
<title>time.stopTimer (33 samples, 0.01%)</title><rect x="1085.5" y="677" width="0.1" height="15.0" fill="rgb(212,182,24)" rx="2" ry="2" />
<text  x="1088.52" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (52 samples, 0.02%)</title><rect x="192.1" y="661" width="0.2" height="15.0" fill="rgb(243,155,17)" rx="2" ry="2" />
<text  x="195.08" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).notifyProtocolGoroutine (164 samples, 0.05%)</title><rect x="65.5" y="661" width="0.6" height="15.0" fill="rgb(226,142,22)" rx="2" ry="2" />
<text  x="68.50" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (96 samples, 0.03%)</title><rect x="13.0" y="645" width="0.4" height="15.0" fill="rgb(226,61,26)" rx="2" ry="2" />
<text  x="16.04" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.newOutgoingSegment (122 samples, 0.04%)</title><rect x="207.3" y="677" width="0.4" height="15.0" fill="rgb(220,185,6)" rx="2" ry="2" />
<text  x="210.27" y="687.5" ></text>
</g>
<g >
<title>runtime.mallocgc (194 samples, 0.06%)</title><rect x="168.0" y="709" width="0.7" height="15.0" fill="rgb(235,187,7)" rx="2" ry="2" />
<text  x="171.03" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (51 samples, 0.02%)</title><rect x="1001.1" y="805" width="0.2" height="15.0" fill="rgb(226,127,23)" rx="2" ry="2" />
<text  x="1004.11" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="700.0" y="677" width="0.1" height="15.0" fill="rgb(242,110,11)" rx="2" ry="2" />
<text  x="702.96" y="687.5" ></text>
</g>
<g >
<title>runtime.ready (34 samples, 0.01%)</title><rect x="1074.1" y="485" width="0.1" height="15.0" fill="rgb(235,208,28)" rx="2" ry="2" />
<text  x="1077.08" y="495.5" ></text>
</g>
<g >
<title>runtime.exitsyscallfast.func1 (60 samples, 0.02%)</title><rect x="842.9" y="789" width="0.2" height="15.0" fill="rgb(228,215,25)" rx="2" ry="2" />
<text  x="845.87" y="799.5" ></text>
</g>
<g >
<title>runtime.findfunc (94 samples, 0.03%)</title><rect x="1135.4" y="757" width="0.3" height="15.0" fill="rgb(229,103,51)" rx="2" ry="2" />
<text  x="1138.35" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,197 samples, 1.29%)</title><rect x="680.4" y="661" width="15.2" height="15.0" fill="rgb(232,34,28)" rx="2" ry="2" />
<text  x="683.43" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*queueDispatcher).dispatchLoop (28,575 samples, 8.76%)</title><rect x="896.0" y="853" width="103.3" height="15.0" fill="rgb(221,200,2)" rx="2" ry="2" />
<text  x="898.99" y="863.5" >gvisor.dev/g..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (373 samples, 0.11%)</title><rect x="211.8" y="213" width="1.3" height="15.0" fill="rgb(225,84,31)" rx="2" ry="2" />
<text  x="214.77" y="223.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).DeliverNetworkPacket (92 samples, 0.03%)</title><rect x="19.7" y="853" width="0.3" height="15.0" fill="rgb(229,127,28)" rx="2" ry="2" />
<text  x="22.66" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*channelCallback).Callback (663 samples, 0.20%)</title><rect x="1001.6" y="789" width="2.4" height="15.0" fill="rgb(219,171,47)" rx="2" ry="2" />
<text  x="1004.65" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn (118 samples, 0.04%)</title><rect x="203.9" y="709" width="0.4" height="15.0" fill="rgb(244,225,52)" rx="2" ry="2" />
<text  x="206.86" y="719.5" ></text>
</g>
<g >
<title>runtime.park_m (513 samples, 0.16%)</title><rect x="1107.5" y="821" width="1.9" height="15.0" fill="rgb(251,171,51)" rx="2" ry="2" />
<text  x="1110.50" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).DeliverNetworkPacket (20,709 samples, 6.35%)</title><rect x="720.3" y="757" width="74.9" height="15.0" fill="rgb(236,211,18)" rx="2" ry="2" />
<text  x="723.33" y="767.5" >gvisor.d..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacket (392 samples, 0.12%)</title><rect x="1027.7" y="645" width="1.4" height="15.0" fill="rgb(223,99,2)" rx="2" ry="2" />
<text  x="1030.65" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (88 samples, 0.03%)</title><rect x="1108.7" y="645" width="0.3" height="15.0" fill="rgb(251,7,25)" rx="2" ry="2" />
<text  x="1111.72" y="655.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc1 (28 samples, 0.01%)</title><rect x="1068.3" y="757" width="0.1" height="15.0" fill="rgb(243,179,42)" rx="2" ry="2" />
<text  x="1071.34" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="1153.4" y="597" width="0.1" height="15.0" fill="rgb(224,200,18)" rx="2" ry="2" />
<text  x="1156.37" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,839 samples, 0.56%)</title><rect x="958.5" y="485" width="6.6" height="15.0" fill="rgb(249,108,48)" rx="2" ry="2" />
<text  x="961.47" y="495.5" ></text>
</g>
<g >
<title>runtime.convT64 (37 samples, 0.01%)</title><rect x="52.4" y="693" width="0.2" height="15.0" fill="rgb(212,107,54)" rx="2" ry="2" />
<text  x="55.45" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (74 samples, 0.02%)</title><rect x="18.4" y="677" width="0.3" height="15.0" fill="rgb(210,4,22)" rx="2" ry="2" />
<text  x="21.40" y="687.5" ></text>
</g>
<g >
<title>runtime.mallocgc (866 samples, 0.27%)</title><rect x="713.8" y="789" width="3.1" height="15.0" fill="rgb(211,56,3)" rx="2" ry="2" />
<text  x="716.82" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (37 samples, 0.01%)</title><rect x="1007.5" y="549" width="0.2" height="15.0" fill="rgb(224,78,33)" rx="2" ry="2" />
<text  x="1010.52" y="559.5" ></text>
</g>
<g >
<title>runtime.nanotime1 (1,439 samples, 0.44%)</title><rect x="23.7" y="869" width="5.2" height="15.0" fill="rgb(208,158,4)" rx="2" ry="2" />
<text  x="26.74" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (243 samples, 0.07%)</title><rect x="1063.9" y="565" width="0.9" height="15.0" fill="rgb(251,146,54)" rx="2" ry="2" />
<text  x="1066.92" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).reserveTupleLocked (300 samples, 0.09%)</title><rect x="1025.3" y="805" width="1.0" height="15.0" fill="rgb(215,136,45)" rx="2" ry="2" />
<text  x="1028.25" y="815.5" ></text>
</g>
<g >
<title>runtime.schedule (47 samples, 0.01%)</title><rect x="16.8" y="725" width="0.1" height="15.0" fill="rgb(213,152,24)" rx="2" ry="2" />
<text  x="19.75" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="1153.0" y="581" width="0.1" height="15.0" fill="rgb(213,4,54)" rx="2" ry="2" />
<text  x="1155.95" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*overlayMountSourceOperations).Revalidate (119 samples, 0.04%)</title><rect x="186.6" y="677" width="0.5" height="15.0" fill="rgb(205,96,9)" rx="2" ry="2" />
<text  x="189.64" y="687.5" ></text>
</g>
<g >
<title>__ext4_get_inode_loc (84 samples, 0.03%)</title><rect x="197.2" y="197" width="0.3" height="15.0" fill="rgb(254,214,19)" rx="2" ry="2" />
<text  x="200.23" y="207.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (567 samples, 0.17%)</title><rect x="987.6" y="149" width="2.1" height="15.0" fill="rgb(240,133,26)" rx="2" ry="2" />
<text  x="990.63" y="159.5" ></text>
</g>
<g >
<title>runtime.park_m (1,194 samples, 0.37%)</title><rect x="698.7" y="821" width="4.4" height="15.0" fill="rgb(238,176,28)" rx="2" ry="2" />
<text  x="701.74" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).offsetForAppend (116 samples, 0.04%)</title><rect x="191.1" y="725" width="0.4" height="15.0" fill="rgb(213,78,24)" rx="2" ry="2" />
<text  x="194.11" y="735.5" ></text>
</g>
<g >
<title>runtime.selectnbsend (50 samples, 0.02%)</title><rect x="1004.2" y="837" width="0.2" height="15.0" fill="rgb(216,10,27)" rx="2" ry="2" />
<text  x="1007.24" y="847.5" ></text>
</g>
<g >
<title>runtime.checkTimers (67 samples, 0.02%)</title><rect x="699.7" y="773" width="0.2" height="15.0" fill="rgb(246,71,3)" rx="2" ry="2" />
<text  x="702.65" y="783.5" ></text>
</g>
<g >
<title>runtime.mallocgc (38 samples, 0.01%)</title><rect x="1036.5" y="613" width="0.2" height="15.0" fill="rgb(206,131,1)" rx="2" ry="2" />
<text  x="1039.52" y="623.5" ></text>
</g>
<g >
<title>runtime.memmove (59 samples, 0.02%)</title><rect x="929.3" y="805" width="0.2" height="15.0" fill="rgb(233,155,44)" rx="2" ry="2" />
<text  x="932.33" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*subprocess).switchToApp (54 samples, 0.02%)</title><rect x="696.0" y="837" width="0.2" height="15.0" fill="rgb(238,61,23)" rx="2" ry="2" />
<text  x="699.03" y="847.5" ></text>
</g>
<g >
<title>runtime.procyield (45 samples, 0.01%)</title><rect x="1153.6" y="709" width="0.2" height="15.0" fill="rgb(214,35,15)" rx="2" ry="2" />
<text  x="1156.63" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).MTU (37 samples, 0.01%)</title><rect x="1009.4" y="725" width="0.1" height="15.0" fill="rgb(247,66,11)" rx="2" ry="2" />
<text  x="1012.39" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (66 samples, 0.02%)</title><rect x="1189.8" y="677" width="0.2" height="15.0" fill="rgb(243,112,12)" rx="2" ry="2" />
<text  x="1192.75" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (805 samples, 0.25%)</title><rect x="819.2" y="629" width="3.0" height="15.0" fill="rgb(235,200,44)" rx="2" ry="2" />
<text  x="822.25" y="639.5" ></text>
</g>
<g >
<title>syscall.Syscall6 (60,758 samples, 18.62%)</title><rect x="231.1" y="773" width="219.7" height="15.0" fill="rgb(235,184,50)" rx="2" ry="2" />
<text  x="234.07" y="783.5" >syscall.Syscall6</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (37 samples, 0.01%)</title><rect x="790.8" y="693" width="0.1" height="15.0" fill="rgb(254,215,42)" rx="2" ry="2" />
<text  x="793.81" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (156 samples, 0.05%)</title><rect x="451.7" y="613" width="0.5" height="15.0" fill="rgb(250,18,9)" rx="2" ry="2" />
<text  x="454.67" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="766.3" y="421" width="0.2" height="15.0" fill="rgb(252,155,37)" rx="2" ry="2" />
<text  x="769.34" y="431.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*processor).start (66 samples, 0.02%)</title><rect x="1107.0" y="869" width="0.3" height="15.0" fill="rgb(248,2,51)" rx="2" ry="2" />
<text  x="1110.01" y="879.5" ></text>
</g>
<g >
<title>runtime.stopm (5,855 samples, 1.79%)</title><rect x="127.5" y="629" width="21.2" height="15.0" fill="rgb(254,211,35)" rx="2" ry="2" />
<text  x="130.49" y="639.5" ></text>
</g>
<g >
<title>ipv4_confirm (308 samples, 0.09%)</title><rect x="992.6" y="309" width="1.1" height="15.0" fill="rgb(208,184,13)" rx="2" ry="2" />
<text  x="995.60" y="319.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.constructAndValidateRoute (350 samples, 0.11%)</title><rect x="1033.1" y="757" width="1.3" height="15.0" fill="rgb(210,200,26)" rx="2" ry="2" />
<text  x="1036.11" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*AtomicRefCount).DecRefWithDestructor (1,446 samples, 0.44%)</title><rect x="63.4" y="741" width="5.3" height="15.0" fill="rgb(247,155,1)" rx="2" ry="2" />
<text  x="66.44" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (60 samples, 0.02%)</title><rect x="176.4" y="597" width="0.3" height="15.0" fill="rgb(243,84,14)" rx="2" ry="2" />
<text  x="179.44" y="607.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (29 samples, 0.01%)</title><rect x="206.9" y="629" width="0.1" height="15.0" fill="rgb(243,146,26)" rx="2" ry="2" />
<text  x="209.90" y="639.5" ></text>
</g>
<g >
<title>runtime.procyield (193 samples, 0.06%)</title><rect x="116.9" y="629" width="0.7" height="15.0" fill="rgb(223,23,43)" rx="2" ry="2" />
<text  x="119.94" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="149.2" y="453" width="0.2" height="15.0" fill="rgb(206,63,19)" rx="2" ry="2" />
<text  x="152.19" y="463.5" ></text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (80 samples, 0.02%)</title><rect x="711.2" y="709" width="0.3" height="15.0" fill="rgb(247,99,39)" rx="2" ry="2" />
<text  x="714.17" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.buildTCPHdr (77 samples, 0.02%)</title><rect x="214.9" y="581" width="0.3" height="15.0" fill="rgb(210,3,52)" rx="2" ry="2" />
<text  x="217.90" y="591.5" ></text>
</g>
<g >
<title>runtime.startlockedm (135 samples, 0.04%)</title><rect x="157.3" y="645" width="0.5" height="15.0" fill="rgb(254,53,26)" rx="2" ry="2" />
<text  x="160.34" y="655.5" ></text>
</g>
<g >
<title>runtime.chansend (42 samples, 0.01%)</title><rect x="1004.2" y="821" width="0.2" height="15.0" fill="rgb(237,99,7)" rx="2" ry="2" />
<text  x="1007.24" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).contextValue (37 samples, 0.01%)</title><rect x="47.6" y="677" width="0.1" height="15.0" fill="rgb(230,26,11)" rx="2" ry="2" />
<text  x="50.56" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*pollEntryList).Remove (127 samples, 0.04%)</title><rect x="169.0" y="741" width="0.4" height="15.0" fill="rgb(206,108,44)" rx="2" ry="2" />
<text  x="171.96" y="751.5" ></text>
</g>
<g >
<title>runtime.usleep (67 samples, 0.02%)</title><rect x="18.8" y="853" width="0.2" height="15.0" fill="rgb(243,26,16)" rx="2" ry="2" />
<text  x="21.79" y="863.5" ></text>
</g>
<g >
<title>runtime.ready (3,225 samples, 0.99%)</title><rect x="746.5" y="565" width="11.7" height="15.0" fill="rgb(207,52,24)" rx="2" ry="2" />
<text  x="749.52" y="575.5" ></text>
</g>
<g >
<title>time.AfterFunc (161 samples, 0.05%)</title><rect x="1010.8" y="741" width="0.6" height="15.0" fill="rgb(252,144,9)" rx="2" ry="2" />
<text  x="1013.83" y="751.5" ></text>
</g>
<g >
<title>__xattr_check_inode (37 samples, 0.01%)</title><rect x="197.5" y="197" width="0.2" height="15.0" fill="rgb(219,140,36)" rx="2" ry="2" />
<text  x="200.54" y="207.5" ></text>
</g>
<g >
<title>runtime.lock2 (97 samples, 0.03%)</title><rect x="844.3" y="741" width="0.3" height="15.0" fill="rgb(232,174,51)" rx="2" ry="2" />
<text  x="847.29" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*dispatcher).queuePacket (83 samples, 0.03%)</title><rect x="742.8" y="661" width="0.3" height="15.0" fill="rgb(223,120,46)" rx="2" ry="2" />
<text  x="745.78" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (335 samples, 0.10%)</title><rect x="1188.5" y="677" width="1.2" height="15.0" fill="rgb(243,50,51)" rx="2" ry="2" />
<text  x="1191.50" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (37 samples, 0.01%)</title><rect x="998.9" y="821" width="0.1" height="15.0" fill="rgb(235,201,42)" rx="2" ry="2" />
<text  x="1001.88" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (284 samples, 0.09%)</title><rect x="114.9" y="597" width="1.0" height="15.0" fill="rgb(217,187,49)" rx="2" ry="2" />
<text  x="117.85" y="607.5" ></text>
</g>
<g >
<title>runtime.(*lfstack).push (154 samples, 0.05%)</title><rect x="1133.2" y="741" width="0.6" height="15.0" fill="rgb(219,180,18)" rx="2" ry="2" />
<text  x="1136.22" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).SwapAnd (76 samples, 0.02%)</title><rect x="1047.6" y="725" width="0.3" height="15.0" fill="rgb(231,52,44)" rx="2" ry="2" />
<text  x="1050.60" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.newSubprocess.func1 (810 samples, 0.25%)</title><rect x="703.8" y="869" width="3.0" height="15.0" fill="rgb(253,222,44)" rx="2" ry="2" />
<text  x="706.84" y="879.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (33 samples, 0.01%)</title><rect x="776.0" y="677" width="0.1" height="15.0" fill="rgb(219,2,26)" rx="2" ry="2" />
<text  x="778.99" y="687.5" ></text>
</g>
<g >
<title>runtime.exitsyscallfast (81 samples, 0.02%)</title><rect x="842.6" y="789" width="0.3" height="15.0" fill="rgb(224,190,11)" rx="2" ry="2" />
<text  x="845.57" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendRaw (28 samples, 0.01%)</title><rect x="1107.1" y="821" width="0.1" height="15.0" fill="rgb(244,159,35)" rx="2" ry="2" />
<text  x="1110.13" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="701.9" y="597" width="0.1" height="15.0" fill="rgb(238,165,41)" rx="2" ry="2" />
<text  x="704.87" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="22.5" y="725" width="0.1" height="15.0" fill="rgb(249,228,10)" rx="2" ry="2" />
<text  x="25.51" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).hasAddress (509 samples, 0.16%)</title><rect x="1035.9" y="741" width="1.9" height="15.0" fill="rgb(206,72,14)" rx="2" ry="2" />
<text  x="1038.93" y="751.5" ></text>
</g>
<g >
<title>runtime.step (132 samples, 0.04%)</title><rect x="1096.5" y="661" width="0.5" height="15.0" fill="rgb(228,127,16)" rx="2" ry="2" />
<text  x="1099.48" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doSyscallEnter (28 samples, 0.01%)</title><rect x="221.2" y="837" width="0.1" height="15.0" fill="rgb(224,66,30)" rx="2" ry="2" />
<text  x="224.19" y="847.5" ></text>
</g>
<g >
<title>runtime.mapiterinit (36 samples, 0.01%)</title><rect x="791.0" y="693" width="0.2" height="15.0" fill="rgb(236,198,4)" rx="2" ry="2" />
<text  x="794.03" y="703.5" ></text>
</g>
<g >
<title>runtime.mallocgc (33 samples, 0.01%)</title><rect x="1005.1" y="821" width="0.1" height="15.0" fill="rgb(250,212,5)" rx="2" ry="2" />
<text  x="1008.11" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendData (860 samples, 0.26%)</title><rect x="179.9" y="677" width="3.1" height="15.0" fill="rgb(208,205,4)" rx="2" ry="2" />
<text  x="182.92" y="687.5" ></text>
</g>
<g >
<title>runtime.wirep (35 samples, 0.01%)</title><rect x="450.4" y="725" width="0.1" height="15.0" fill="rgb(228,168,20)" rx="2" ry="2" />
<text  x="453.38" y="735.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_faststr (73 samples, 0.02%)</title><rect x="788.9" y="677" width="0.2" height="15.0" fill="rgb(246,116,46)" rx="2" ry="2" />
<text  x="791.86" y="687.5" ></text>
</g>
<g >
<title>runtime.exitsyscall (582 samples, 0.18%)</title><rect x="448.5" y="757" width="2.1" height="15.0" fill="rgb(213,191,48)" rx="2" ry="2" />
<text  x="451.50" y="767.5" ></text>
</g>
<g >
<title>runtime.resetspinning (54 samples, 0.02%)</title><rect x="1109.1" y="789" width="0.2" height="15.0" fill="rgb(238,9,22)" rx="2" ry="2" />
<text  x="1112.11" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyInBytes (119 samples, 0.04%)</title><rect x="56.7" y="693" width="0.4" height="15.0" fill="rgb(249,90,32)" rx="2" ry="2" />
<text  x="59.68" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacketBuffer (254 samples, 0.08%)</title><rect x="1091.1" y="677" width="0.9" height="15.0" fill="rgb(206,103,38)" rx="2" ry="2" />
<text  x="1094.12" y="687.5" ></text>
</g>
<g >
<title>runtime.stoplockedm (165 samples, 0.05%)</title><rect x="451.6" y="693" width="0.6" height="15.0" fill="rgb(254,109,27)" rx="2" ry="2" />
<text  x="454.64" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).Fetch (322 samples, 0.10%)</title><rect x="1045.6" y="853" width="1.2" height="15.0" fill="rgb(221,89,25)" rx="2" ry="2" />
<text  x="1048.64" y="863.5" ></text>
</g>
<g >
<title>runtime.doaddtimer (63 samples, 0.02%)</title><rect x="1082.2" y="597" width="0.2" height="15.0" fill="rgb(210,195,48)" rx="2" ry="2" />
<text  x="1085.16" y="607.5" ></text>
</g>
<g >
<title>runtime.stoplockedm (426 samples, 0.13%)</title><rect x="705.2" y="773" width="1.5" height="15.0" fill="rgb(214,219,24)" rx="2" ry="2" />
<text  x="708.16" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/usermem.(*IOSequenceReadWriter).Write (236 samples, 0.07%)</title><rect x="176.1" y="677" width="0.9" height="15.0" fill="rgb(235,118,38)" rx="2" ry="2" />
<text  x="179.12" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (54 samples, 0.02%)</title><rect x="18.0" y="725" width="0.2" height="15.0" fill="rgb(223,74,33)" rx="2" ry="2" />
<text  x="20.99" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (277 samples, 0.08%)</title><rect x="703.9" y="741" width="1.0" height="15.0" fill="rgb(244,118,48)" rx="2" ry="2" />
<text  x="706.88" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (48 samples, 0.01%)</title><rect x="1051.6" y="709" width="0.2" height="15.0" fill="rgb(207,226,19)" rx="2" ry="2" />
<text  x="1054.63" y="719.5" ></text>
</g>
<g >
<title>runtime.mallocgc (87 samples, 0.03%)</title><rect x="66.6" y="677" width="0.3" height="15.0" fill="rgb(220,146,49)" rx="2" ry="2" />
<text  x="69.59" y="687.5" ></text>
</g>
<g >
<title>runtime.mallocgc (36 samples, 0.01%)</title><rect x="216.4" y="693" width="0.2" height="15.0" fill="rgb(226,119,2)" rx="2" ry="2" />
<text  x="219.43" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).syscallIgnoreInterrupt (340 samples, 0.10%)</title><rect x="703.8" y="837" width="1.3" height="15.0" fill="rgb(230,176,29)" rx="2" ry="2" />
<text  x="706.85" y="847.5" ></text>
</g>
<g >
<title>nft_do_chain_ipv4 (40 samples, 0.01%)</title><rect x="992.2" y="181" width="0.1" height="15.0" fill="rgb(229,67,50)" rx="2" ry="2" />
<text  x="995.19" y="191.5" ></text>
</g>
<g >
<title>runtime/trace.StartRegion (34 samples, 0.01%)</title><rect x="696.2" y="837" width="0.2" height="15.0" fill="rgb(240,59,16)" rx="2" ry="2" />
<text  x="699.25" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (82 samples, 0.03%)</title><rect x="1062.2" y="629" width="0.3" height="15.0" fill="rgb(230,111,48)" rx="2" ry="2" />
<text  x="1065.20" y="639.5" ></text>
</g>
<g >
<title>runtime.futex (63 samples, 0.02%)</title><rect x="845.7" y="821" width="0.3" height="15.0" fill="rgb(231,34,29)" rx="2" ry="2" />
<text  x="848.74" y="831.5" ></text>
</g>
<g >
<title>runtime.(*stackScanState).addObject (296 samples, 0.09%)</title><rect x="1138.0" y="725" width="1.0" height="15.0" fill="rgb(220,44,4)" rx="2" ry="2" />
<text  x="1140.96" y="735.5" ></text>
</g>
<g >
<title>sync/atomic.(*Value).Store (31 samples, 0.01%)</title><rect x="45.5" y="677" width="0.1" height="15.0" fill="rgb(214,13,41)" rx="2" ry="2" />
<text  x="48.51" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (16,711 samples, 5.12%)</title><rect x="937.7" y="709" width="60.4" height="15.0" fill="rgb(229,194,8)" rx="2" ry="2" />
<text  x="940.70" y="719.5" >[[kern..</text>
</g>
<g >
<title>runtime.newobject (34 samples, 0.01%)</title><rect x="1025.1" y="773" width="0.1" height="15.0" fill="rgb(214,156,49)" rx="2" ry="2" />
<text  x="1028.07" y="783.5" ></text>
</g>
<g >
<title>runtime.mallocgc (109 samples, 0.03%)</title><rect x="1011.5" y="741" width="0.4" height="15.0" fill="rgb(248,21,1)" rx="2" ry="2" />
<text  x="1014.52" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (35 samples, 0.01%)</title><rect x="741.0" y="677" width="0.1" height="15.0" fill="rgb(218,138,14)" rx="2" ry="2" />
<text  x="743.95" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn.func1 (73 samples, 0.02%)</title><rect x="184.0" y="677" width="0.3" height="15.0" fill="rgb(245,23,40)" rx="2" ry="2" />
<text  x="187.00" y="687.5" ></text>
</g>
<g >
<title>runtime.mallocgc (261 samples, 0.08%)</title><rect x="849.4" y="805" width="0.9" height="15.0" fill="rgb(244,65,7)" rx="2" ry="2" />
<text  x="852.37" y="815.5" ></text>
</g>
<g >
<title>syscall.Syscall6 (40 samples, 0.01%)</title><rect x="21.6" y="565" width="0.1" height="15.0" fill="rgb(206,165,50)" rx="2" ry="2" />
<text  x="24.59" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (79 samples, 0.02%)</title><rect x="1007.4" y="645" width="0.3" height="15.0" fill="rgb(228,121,39)" rx="2" ry="2" />
<text  x="1010.37" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (69 samples, 0.02%)</title><rect x="117.8" y="549" width="0.3" height="15.0" fill="rgb(248,33,21)" rx="2" ry="2" />
<text  x="120.84" y="559.5" ></text>
</g>
<g >
<title>runtime.acquirep (115 samples, 0.04%)</title><rect x="843.9" y="741" width="0.4" height="15.0" fill="rgb(235,67,35)" rx="2" ry="2" />
<text  x="846.88" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (377 samples, 0.12%)</title><rect x="1125.4" y="629" width="1.4" height="15.0" fill="rgb(254,118,43)" rx="2" ry="2" />
<text  x="1128.43" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (116 samples, 0.04%)</title><rect x="702.4" y="661" width="0.5" height="15.0" fill="rgb(210,210,11)" rx="2" ry="2" />
<text  x="705.44" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (153 samples, 0.05%)</title><rect x="451.7" y="549" width="0.5" height="15.0" fill="rgb(236,208,30)" rx="2" ry="2" />
<text  x="454.68" y="559.5" ></text>
</g>
<g >
<title>runtime.acquireSudog (35 samples, 0.01%)</title><rect x="1004.6" y="805" width="0.2" height="15.0" fill="rgb(252,69,17)" rx="2" ry="2" />
<text  x="1007.64" y="815.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (176 samples, 0.05%)</title><rect x="777.7" y="709" width="0.6" height="15.0" fill="rgb(236,225,36)" rx="2" ry="2" />
<text  x="780.69" y="719.5" ></text>
</g>
<g >
<title>runtime.mcall (546 samples, 0.17%)</title><rect x="21.8" y="869" width="1.9" height="15.0" fill="rgb(211,90,39)" rx="2" ry="2" />
<text  x="24.76" y="879.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (124 samples, 0.04%)</title><rect x="733.9" y="677" width="0.5" height="15.0" fill="rgb(216,195,5)" rx="2" ry="2" />
<text  x="736.95" y="687.5" ></text>
</g>
<g >
<title>runtime.notesleep (417 samples, 0.13%)</title><rect x="705.2" y="757" width="1.5" height="15.0" fill="rgb(229,75,31)" rx="2" ry="2" />
<text  x="708.19" y="767.5" ></text>
</g>
<g >
<title>runtime.closechan (54 samples, 0.02%)</title><rect x="1105.6" y="853" width="0.2" height="15.0" fill="rgb(223,72,6)" rx="2" ry="2" />
<text  x="1108.63" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,235 samples, 0.99%)</title><rect x="136.7" y="533" width="11.7" height="15.0" fill="rgb(219,197,45)" rx="2" ry="2" />
<text  x="139.72" y="543.5" ></text>
</g>
<g >
<title>runtime.checkTimers (163 samples, 0.05%)</title><rect x="1045.9" y="773" width="0.6" height="15.0" fill="rgb(237,118,17)" rx="2" ry="2" />
<text  x="1048.90" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*protocol).ParseAddresses (220 samples, 0.07%)</title><rect x="781.0" y="741" width="0.8" height="15.0" fill="rgb(233,101,43)" rx="2" ry="2" />
<text  x="784.04" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (41 samples, 0.01%)</title><rect x="1107.3" y="853" width="0.1" height="15.0" fill="rgb(213,114,52)" rx="2" ry="2" />
<text  x="1110.29" y="863.5" ></text>
</g>
<g >
<title>time.startTimer (79 samples, 0.02%)</title><rect x="1011.1" y="725" width="0.3" height="15.0" fill="rgb(235,50,21)" rx="2" ry="2" />
<text  x="1014.12" y="735.5" ></text>
</g>
<g >
<title>runtime.usleep (451 samples, 0.14%)</title><rect x="14.2" y="757" width="1.7" height="15.0" fill="rgb(226,192,47)" rx="2" ry="2" />
<text  x="17.23" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="962.8" y="405" width="0.1" height="15.0" fill="rgb(225,210,54)" rx="2" ry="2" />
<text  x="965.76" y="415.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).destroy (63 samples, 0.02%)</title><rect x="64.0" y="597" width="0.3" height="15.0" fill="rgb(211,43,49)" rx="2" ry="2" />
<text  x="67.04" y="607.5" ></text>
</g>
<g >
<title>runtime.funcdata (47 samples, 0.01%)</title><rect x="1139.7" y="709" width="0.1" height="15.0" fill="rgb(237,133,53)" rx="2" ry="2" />
<text  x="1142.67" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (245 samples, 0.08%)</title><rect x="1125.9" y="597" width="0.9" height="15.0" fill="rgb(232,64,50)" rx="2" ry="2" />
<text  x="1128.91" y="607.5" ></text>
</g>
<g >
<title>syscall.Syscall6 (116 samples, 0.04%)</title><rect x="695.6" y="821" width="0.4" height="15.0" fill="rgb(235,198,16)" rx="2" ry="2" />
<text  x="698.61" y="831.5" ></text>
</g>
<g >
<title>runtime.runqput (62 samples, 0.02%)</title><rect x="747.4" y="549" width="0.2" height="15.0" fill="rgb(249,188,24)" rx="2" ry="2" />
<text  x="750.37" y="559.5" ></text>
</g>
<g >
<title>runtime.makechan (67 samples, 0.02%)</title><rect x="1079.6" y="645" width="0.3" height="15.0" fill="rgb(252,212,22)" rx="2" ry="2" />
<text  x="1082.64" y="655.5" ></text>
</g>
<g >
<title>runtime.pcdatavalue (264 samples, 0.08%)</title><rect x="1148.9" y="693" width="1.0" height="15.0" fill="rgb(229,30,34)" rx="2" ry="2" />
<text  x="1151.90" y="703.5" ></text>
</g>
<g >
<title>runtime.stopm (646 samples, 0.20%)</title><rect x="1062.5" y="741" width="2.3" height="15.0" fill="rgb(223,153,0)" rx="2" ry="2" />
<text  x="1065.50" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*context).Switch (67 samples, 0.02%)</title><rect x="696.4" y="853" width="0.2" height="15.0" fill="rgb(235,48,48)" rx="2" ry="2" />
<text  x="699.37" y="863.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (788 samples, 0.24%)</title><rect x="770.1" y="645" width="2.8" height="15.0" fill="rgb(235,171,47)" rx="2" ry="2" />
<text  x="773.10" y="655.5" ></text>
</g>
<g >
<title>runtime.mallocgc (144 samples, 0.04%)</title><rect x="777.8" y="693" width="0.5" height="15.0" fill="rgb(207,61,30)" rx="2" ry="2" />
<text  x="780.80" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*endpoint).Attach.func1 (52,230 samples, 16.01%)</title><rect x="706.9" y="869" width="188.8" height="15.0" fill="rgb(239,1,43)" rx="2" ry="2" />
<text  x="709.86" y="879.5" >gvisor.dev/gvisor/pkg/tc..</text>
</g>
<g >
<title>runtime.exitsyscall (28 samples, 0.01%)</title><rect x="230.6" y="773" width="0.1" height="15.0" fill="rgb(213,132,0)" rx="2" ry="2" />
<text  x="233.56" y="783.5" ></text>
</g>
<g >
<title>nf_ct_deliver_cached_events (38 samples, 0.01%)</title><rect x="992.4" y="181" width="0.1" height="15.0" fill="rgb(224,132,40)" rx="2" ry="2" />
<text  x="995.37" y="191.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls.WaitEpoll (37 samples, 0.01%)</title><rect x="43.1" y="773" width="0.2" height="15.0" fill="rgb(249,131,14)" rx="2" ry="2" />
<text  x="46.14" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (72 samples, 0.02%)</title><rect x="228.8" y="789" width="0.2" height="15.0" fill="rgb(251,135,50)" rx="2" ry="2" />
<text  x="231.77" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*endpointsByNIC).handlePacket (7,927 samples, 2.43%)</title><rect x="741.1" y="677" width="28.6" height="15.0" fill="rgb(226,225,20)" rx="2" ry="2" />
<text  x="744.08" y="687.5" >gv..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (6,403 samples, 1.96%)</title><rect x="871.7" y="725" width="23.2" height="15.0" fill="rgb(226,164,23)" rx="2" ry="2" />
<text  x="874.73" y="735.5" >[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).protocolMainLoop (17,042 samples, 5.22%)</title><rect x="1045.3" y="869" width="61.6" height="15.0" fill="rgb(217,124,15)" rx="2" ry="2" />
<text  x="1048.29" y="879.5" >gvisor..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (10,828 samples, 3.32%)</title><rect x="598.8" y="693" width="39.2" height="15.0" fill="rgb(233,99,18)" rx="2" ry="2" />
<text  x="601.85" y="703.5" >[[k..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (91 samples, 0.03%)</title><rect x="228.1" y="789" width="0.3" height="15.0" fill="rgb(247,71,53)" rx="2" ry="2" />
<text  x="231.11" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).WritePacket (736 samples, 0.23%)</title><rect x="1090.7" y="757" width="2.7" height="15.0" fill="rgb(231,112,36)" rx="2" ry="2" />
<text  x="1093.74" y="767.5" ></text>
</g>
<g >
<title>runtime.greyobject (1,449 samples, 0.44%)</title><rect x="1178.0" y="805" width="5.3" height="15.0" fill="rgb(242,111,35)" rx="2" ry="2" />
<text  x="1181.04" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (55,024 samples, 16.86%)</title><rect x="247.7" y="741" width="199.0" height="15.0" fill="rgb(246,107,5)" rx="2" ry="2" />
<text  x="250.67" y="751.5" >[[kernel.kallsyms]]</text>
</g>
<g >
<title>[[kernel.kallsyms]] (325 samples, 0.10%)</title><rect x="995.5" y="453" width="1.1" height="15.0" fill="rgb(242,68,21)" rx="2" ry="2" />
<text  x="998.46" y="463.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.unrolledCalculateChecksum (207 samples, 0.06%)</title><rect x="731.3" y="693" width="0.8" height="15.0" fill="rgb(250,131,12)" rx="2" ry="2" />
<text  x="734.31" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).addIPHeader (46 samples, 0.01%)</title><rect x="1073.3" y="677" width="0.1" height="15.0" fill="rgb(222,55,38)" rx="2" ry="2" />
<text  x="1076.28" y="687.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (38 samples, 0.01%)</title><rect x="1085.3" y="645" width="0.1" height="15.0" fill="rgb(251,20,1)" rx="2" ry="2" />
<text  x="1088.31" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.Writer.WriteFromBlocks-fm (2,707 samples, 0.83%)</title><rect x="192.7" y="629" width="9.7" height="15.0" fill="rgb(240,22,35)" rx="2" ry="2" />
<text  x="195.65" y="639.5" ></text>
</g>
<g >
<title>runtime.isSystemGoroutine (37 samples, 0.01%)</title><rect x="114.2" y="517" width="0.2" height="15.0" fill="rgb(227,0,12)" rx="2" ry="2" />
<text  x="117.24" y="527.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/usermem.CopyStringIn (269 samples, 0.08%)</title><rect x="183.8" y="725" width="1.0" height="15.0" fill="rgb(231,97,13)" rx="2" ry="2" />
<text  x="186.83" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (353 samples, 0.11%)</title><rect x="1186.3" y="757" width="1.3" height="15.0" fill="rgb(215,17,44)" rx="2" ry="2" />
<text  x="1189.34" y="767.5" ></text>
</g>
<g >
<title>tcp_error (206 samples, 0.06%)</title><rect x="962.1" y="453" width="0.8" height="15.0" fill="rgb(234,63,45)" rx="2" ry="2" />
<text  x="965.14" y="463.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (149 samples, 0.05%)</title><rect x="1187.1" y="613" width="0.5" height="15.0" fill="rgb(245,103,53)" rx="2" ry="2" />
<text  x="1190.08" y="623.5" ></text>
</g>
<g >
<title>runtime.siftdownTimer (38 samples, 0.01%)</title><rect x="96.9" y="597" width="0.2" height="15.0" fill="rgb(239,145,8)" rx="2" ry="2" />
<text  x="99.95" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).WritePacket (1,517 samples, 0.46%)</title><rect x="209.4" y="581" width="5.5" height="15.0" fill="rgb(253,139,4)" rx="2" ry="2" />
<text  x="212.40" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (51 samples, 0.02%)</title><rect x="1109.1" y="709" width="0.2" height="15.0" fill="rgb(253,198,26)" rx="2" ry="2" />
<text  x="1112.12" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,135 samples, 0.96%)</title><rect x="137.1" y="517" width="11.3" height="15.0" fill="rgb(252,85,29)" rx="2" ry="2" />
<text  x="140.08" y="527.5" ></text>
</g>
<g >
<title>runtime.gcDrainN (48 samples, 0.01%)</title><rect x="929.1" y="725" width="0.2" height="15.0" fill="rgb(229,103,44)" rx="2" ry="2" />
<text  x="932.08" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,756 samples, 1.76%)</title><rect x="674.8" y="677" width="20.8" height="15.0" fill="rgb(240,26,39)" rx="2" ry="2" />
<text  x="677.79" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/syserr.(*Error).ToError (61 samples, 0.02%)</title><rect x="57.6" y="741" width="0.2" height="15.0" fill="rgb(243,178,42)" rx="2" ry="2" />
<text  x="60.58" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).DeliverOutboundPacket (40 samples, 0.01%)</title><rect x="181.1" y="405" width="0.2" height="15.0" fill="rgb(235,179,50)" rx="2" ry="2" />
<text  x="184.12" y="415.5" ></text>
</g>
<g >
<title>runtime.startm (4,958 samples, 1.52%)</title><rect x="823.6" y="757" width="18.0" height="15.0" fill="rgb(250,210,20)" rx="2" ry="2" />
<text  x="826.64" y="767.5" ></text>
</g>
<g >
<title>nf_nat_ipv4_local_fn (36 samples, 0.01%)</title><rect x="992.1" y="181" width="0.1" height="15.0" fill="rgb(206,68,23)" rx="2" ry="2" />
<text  x="995.06" y="191.5" ></text>
</g>
<g >
<title>runtime.casgstatus (36 samples, 0.01%)</title><rect x="842.4" y="789" width="0.2" height="15.0" fill="rgb(226,182,8)" rx="2" ry="2" />
<text  x="845.44" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (189 samples, 0.06%)</title><rect x="996.0" y="437" width="0.6" height="15.0" fill="rgb(225,136,44)" rx="2" ry="2" />
<text  x="998.95" y="447.5" ></text>
</g>
<g >
<title>runtime.stackcacherefill (254 samples, 0.08%)</title><rect x="1097.9" y="725" width="0.9" height="15.0" fill="rgb(226,185,22)" rx="2" ry="2" />
<text  x="1100.88" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (397 samples, 0.12%)</title><rect x="705.3" y="597" width="1.4" height="15.0" fill="rgb(239,162,31)" rx="2" ry="2" />
<text  x="708.26" y="607.5" ></text>
</g>
<g >
<title>runtime.notewakeup (59 samples, 0.02%)</title><rect x="65.8" y="533" width="0.2" height="15.0" fill="rgb(253,3,53)" rx="2" ry="2" />
<text  x="68.76" y="543.5" ></text>
</g>
<g >
<title>runtime.assertI2I2 (45 samples, 0.01%)</title><rect x="1038.5" y="741" width="0.1" height="15.0" fill="rgb(220,132,16)" rx="2" ry="2" />
<text  x="1041.46" y="751.5" ></text>
</g>
<g >
<title>type..eq.gvisor.dev/gvisor/pkg/tcpip/stack.TransportEndpointID (28 samples, 0.01%)</title><rect x="772.7" y="629" width="0.1" height="15.0" fill="rgb(229,199,25)" rx="2" ry="2" />
<text  x="775.73" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*MountNamespace).resolve (133 samples, 0.04%)</title><rect x="189.0" y="709" width="0.5" height="15.0" fill="rgb(238,48,24)" rx="2" ry="2" />
<text  x="192.03" y="719.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (90 samples, 0.03%)</title><rect x="19.7" y="677" width="0.3" height="15.0" fill="rgb(209,227,8)" rx="2" ry="2" />
<text  x="22.66" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.AddressMask.Prefix (89 samples, 0.03%)</title><rect x="786.1" y="661" width="0.3" height="15.0" fill="rgb(207,208,42)" rx="2" ry="2" />
<text  x="789.08" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="1150.9" y="613" width="0.1" height="15.0" fill="rgb(220,196,36)" rx="2" ry="2" />
<text  x="1153.87" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).WritePacket (35 samples, 0.01%)</title><rect x="181.3" y="421" width="0.1" height="15.0" fill="rgb(240,12,22)" rx="2" ry="2" />
<text  x="184.27" y="431.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*endpointsByNIC).handlePacket (92 samples, 0.03%)</title><rect x="19.7" y="773" width="0.3" height="15.0" fill="rgb(246,64,6)" rx="2" ry="2" />
<text  x="22.66" y="783.5" ></text>
</g>
<g >
<title>runtime.newstack (422 samples, 0.13%)</title><rect x="1183.5" y="869" width="1.6" height="15.0" fill="rgb(217,135,44)" rx="2" ry="2" />
<text  x="1186.54" y="879.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).ptraceSyscallEnter (82 samples, 0.03%)</title><rect x="220.3" y="805" width="0.3" height="15.0" fill="rgb(237,80,36)" rx="2" ry="2" />
<text  x="223.30" y="815.5" ></text>
</g>
<g >
<title>runtime.memmove (85 samples, 0.03%)</title><rect x="75.2" y="661" width="0.3" height="15.0" fill="rgb(254,217,54)" rx="2" ry="2" />
<text  x="78.24" y="671.5" ></text>
</g>
<g >
<title>runtime.scanobject (7,848 samples, 2.41%)</title><rect x="1154.9" y="821" width="28.4" height="15.0" fill="rgb(242,211,13)" rx="2" ry="2" />
<text  x="1157.90" y="831.5" >ru..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*receiver).consumeSegment (1,677 samples, 0.51%)</title><rect x="1071.0" y="789" width="6.0" height="15.0" fill="rgb(213,128,2)" rx="2" ry="2" />
<text  x="1073.98" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).EventUnregister (135 samples, 0.04%)</title><rect x="170.3" y="741" width="0.5" height="15.0" fill="rgb(238,170,13)" rx="2" ry="2" />
<text  x="173.35" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (64 samples, 0.02%)</title><rect x="22.4" y="757" width="0.2" height="15.0" fill="rgb(240,203,30)" rx="2" ry="2" />
<text  x="25.38" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (406 samples, 0.12%)</title><rect x="211.6" y="245" width="1.5" height="15.0" fill="rgb(208,105,42)" rx="2" ry="2" />
<text  x="214.65" y="255.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*threadPool).lookupOrCreate (768 samples, 0.24%)</title><rect x="451.2" y="805" width="2.8" height="15.0" fill="rgb(216,98,5)" rx="2" ry="2" />
<text  x="454.24" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,411 samples, 0.43%)</title><rect x="836.1" y="581" width="5.1" height="15.0" fill="rgb(215,91,36)" rx="2" ry="2" />
<text  x="839.14" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,280 samples, 0.70%)</title><rect x="140.2" y="453" width="8.2" height="15.0" fill="rgb(223,189,52)" rx="2" ry="2" />
<text  x="143.17" y="463.5" ></text>
</g>
<g >
<title>runtime.checkTimers (647 samples, 0.20%)</title><rect x="1116.5" y="805" width="2.3" height="15.0" fill="rgb(250,217,46)" rx="2" ry="2" />
<text  x="1119.46" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).wait (282 samples, 0.09%)</title><rect x="703.9" y="805" width="1.0" height="15.0" fill="rgb(242,40,35)" rx="2" ry="2" />
<text  x="706.86" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (290 samples, 0.09%)</title><rect x="1122.1" y="677" width="1.0" height="15.0" fill="rgb(215,204,25)" rx="2" ry="2" />
<text  x="1125.10" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,231 samples, 1.30%)</title><rect x="806.9" y="741" width="15.3" height="15.0" fill="rgb(218,99,51)" rx="2" ry="2" />
<text  x="809.86" y="751.5" ></text>
</g>
<g >
<title>fmt.(*pp).printArg (104 samples, 0.03%)</title><rect x="48.7" y="661" width="0.4" height="15.0" fill="rgb(223,40,39)" rx="2" ry="2" />
<text  x="51.73" y="671.5" ></text>
</g>
<g >
<title>time.startTimer (270 samples, 0.08%)</title><rect x="1048.8" y="693" width="1.0" height="15.0" fill="rgb(240,38,15)" rx="2" ry="2" />
<text  x="1051.78" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (73 samples, 0.02%)</title><rect x="967.0" y="405" width="0.2" height="15.0" fill="rgb(216,35,24)" rx="2" ry="2" />
<text  x="969.96" y="415.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (154 samples, 0.05%)</title><rect x="1049.2" y="613" width="0.5" height="15.0" fill="rgb(210,109,53)" rx="2" ry="2" />
<text  x="1052.19" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1173.1" y="741" width="0.1" height="15.0" fill="rgb(233,119,46)" rx="2" ry="2" />
<text  x="1176.11" y="751.5" ></text>
</g>
<g >
<title>runtime.newobject (69 samples, 0.02%)</title><rect x="182.6" y="565" width="0.3" height="15.0" fill="rgb(208,138,42)" rx="2" ry="2" />
<text  x="185.62" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (203 samples, 0.06%)</title><rect x="1082.7" y="549" width="0.8" height="15.0" fill="rgb(227,94,38)" rx="2" ry="2" />
<text  x="1085.73" y="559.5" ></text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (89 samples, 0.03%)</title><rect x="716.9" y="789" width="0.4" height="15.0" fill="rgb(222,171,5)" rx="2" ry="2" />
<text  x="719.95" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).init (523 samples, 0.16%)</title><rect x="1047.9" y="725" width="1.9" height="15.0" fill="rgb(227,5,34)" rx="2" ry="2" />
<text  x="1050.88" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress (210 samples, 0.06%)</title><rect x="1034.7" y="725" width="0.7" height="15.0" fill="rgb(226,200,32)" rx="2" ry="2" />
<text  x="1037.68" y="735.5" ></text>
</g>
<g >
<title>runtime.futex (365 samples, 0.11%)</title><rect x="1186.3" y="773" width="1.3" height="15.0" fill="rgb(238,194,10)" rx="2" ry="2" />
<text  x="1189.30" y="783.5" ></text>
</g>
<g >
<title>runtime.mcall (2,748 samples, 0.84%)</title><rect x="1056.9" y="805" width="9.9" height="15.0" fill="rgb(252,13,52)" rx="2" ry="2" />
<text  x="1059.87" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacket (223 samples, 0.07%)</title><rect x="1073.6" y="613" width="0.8" height="15.0" fill="rgb(239,83,3)" rx="2" ry="2" />
<text  x="1076.64" y="623.5" ></text>
</g>
<g >
<title>runtime.newproc.func1 (151 samples, 0.05%)</title><rect x="1043.7" y="805" width="0.5" height="15.0" fill="rgb(225,112,23)" rx="2" ry="2" />
<text  x="1046.70" y="815.5" ></text>
</g>
<g >
<title>runtime.futex (51 samples, 0.02%)</title><rect x="1109.1" y="725" width="0.2" height="15.0" fill="rgb(221,116,51)" rx="2" ry="2" />
<text  x="1112.12" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (39 samples, 0.01%)</title><rect x="1035.2" y="693" width="0.2" height="15.0" fill="rgb(216,119,22)" rx="2" ry="2" />
<text  x="1038.23" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindRoute.func1 (485 samples, 0.15%)</title><rect x="1032.6" y="773" width="1.8" height="15.0" fill="rgb(209,82,3)" rx="2" ry="2" />
<text  x="1035.64" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (125 samples, 0.04%)</title><rect x="117.6" y="613" width="0.5" height="15.0" fill="rgb(206,133,18)" rx="2" ry="2" />
<text  x="120.63" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="848.0" y="645" width="0.1" height="15.0" fill="rgb(220,27,0)" rx="2" ry="2" />
<text  x="850.99" y="655.5" ></text>
</g>
<g >
<title>runtime.resetspinning (130 samples, 0.04%)</title><rect x="18.2" y="805" width="0.5" height="15.0" fill="rgb(228,201,45)" rx="2" ry="2" />
<text  x="21.20" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).ModerateRecvBuf (36 samples, 0.01%)</title><rect x="175.6" y="709" width="0.2" height="15.0" fill="rgb(220,26,28)" rx="2" ry="2" />
<text  x="178.65" y="719.5" ></text>
</g>
<g >
<title>runtime.futex (32 samples, 0.01%)</title><rect x="1129.6" y="757" width="0.1" height="15.0" fill="rgb(229,14,4)" rx="2" ry="2" />
<text  x="1132.55" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).RecordSocket (292 samples, 0.09%)</title><rect x="45.3" y="725" width="1.0" height="15.0" fill="rgb(206,112,32)" rx="2" ry="2" />
<text  x="48.25" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="848.0" y="661" width="0.1" height="15.0" fill="rgb(207,198,12)" rx="2" ry="2" />
<text  x="850.96" y="671.5" ></text>
</g>
<g >
<title>runtime.futex (58 samples, 0.02%)</title><rect x="28.9" y="805" width="0.3" height="15.0" fill="rgb(244,192,44)" rx="2" ry="2" />
<text  x="31.94" y="815.5" ></text>
</g>
<g >
<title>type..hash.gvisor.dev/gvisor/pkg/tcpip/stack.TransportEndpointID (32 samples, 0.01%)</title><rect x="772.8" y="629" width="0.1" height="15.0" fill="rgb(241,205,33)" rx="2" ry="2" />
<text  x="775.83" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (78 samples, 0.02%)</title><rect x="206.0" y="565" width="0.3" height="15.0" fill="rgb(233,91,42)" rx="2" ry="2" />
<text  x="208.98" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*rackControl).updateRACKReorderWindow (35 samples, 0.01%)</title><rect x="1087.7" y="789" width="0.1" height="15.0" fill="rgb(240,138,50)" rx="2" ry="2" />
<text  x="1090.71" y="799.5" ></text>
</g>
<g >
<title>runtime.startm (61 samples, 0.02%)</title><rect x="1051.1" y="661" width="0.3" height="15.0" fill="rgb(250,148,23)" rx="2" ry="2" />
<text  x="1054.13" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (730 samples, 0.22%)</title><rect x="124.7" y="485" width="2.7" height="15.0" fill="rgb(247,170,33)" rx="2" ry="2" />
<text  x="127.73" y="495.5" ></text>
</g>
<g >
<title>ovl_xattr_get (470 samples, 0.14%)</title><rect x="196.2" y="309" width="1.7" height="15.0" fill="rgb(244,84,30)" rx="2" ry="2" />
<text  x="199.23" y="319.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.RecvFrom (56 samples, 0.02%)</title><rect x="219.6" y="789" width="0.2" height="15.0" fill="rgb(207,216,14)" rx="2" ry="2" />
<text  x="222.59" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Shutdown (1,170 samples, 0.36%)</title><rect x="179.4" y="773" width="4.2" height="15.0" fill="rgb(224,166,30)" rx="2" ry="2" />
<text  x="182.40" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).IsBroadcast (174 samples, 0.05%)</title><rect x="735.0" y="661" width="0.6" height="15.0" fill="rgb(218,34,21)" rx="2" ry="2" />
<text  x="737.99" y="671.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (43 samples, 0.01%)</title><rect x="171.7" y="693" width="0.1" height="15.0" fill="rgb(214,96,17)" rx="2" ry="2" />
<text  x="174.67" y="703.5" ></text>
</g>
<g >
<title>runtime.castogscanstatus (33 samples, 0.01%)</title><rect x="1154.6" y="773" width="0.2" height="15.0" fill="rgb(250,11,6)" rx="2" ry="2" />
<text  x="1157.63" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (29 samples, 0.01%)</title><rect x="181.1" y="389" width="0.1" height="15.0" fill="rgb(221,81,3)" rx="2" ry="2" />
<text  x="184.13" y="399.5" ></text>
</g>
<g >
<title>runtime.mcall (55 samples, 0.02%)</title><rect x="1004.8" y="805" width="0.2" height="15.0" fill="rgb(234,2,33)" rx="2" ry="2" />
<text  x="1007.79" y="815.5" ></text>
</g>
<g >
<title>runtime.mapiterinit (267 samples, 0.08%)</title><rect x="789.1" y="677" width="1.0" height="15.0" fill="rgb(214,3,1)" rx="2" ry="2" />
<text  x="792.13" y="687.5" ></text>
</g>
<g >
<title>runtime.pcvalue (262 samples, 0.08%)</title><rect x="1016.9" y="709" width="0.9" height="15.0" fill="rgb(209,0,46)" rx="2" ry="2" />
<text  x="1019.86" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (57 samples, 0.02%)</title><rect x="452.5" y="789" width="0.2" height="15.0" fill="rgb(231,161,10)" rx="2" ry="2" />
<text  x="455.50" y="799.5" ></text>
</g>
<g >
<title>runtime.mallocgc (40 samples, 0.01%)</title><rect x="181.9" y="453" width="0.1" height="15.0" fill="rgb(216,92,46)" rx="2" ry="2" />
<text  x="184.86" y="463.5" ></text>
</g>
<g >
<title>runtime.newobject (247 samples, 0.08%)</title><rect x="50.4" y="677" width="0.9" height="15.0" fill="rgb(212,220,32)" rx="2" ry="2" />
<text  x="53.39" y="687.5" ></text>
</g>
<g >
<title>ipv4_helper (29 samples, 0.01%)</title><rect x="993.7" y="309" width="0.1" height="15.0" fill="rgb(234,222,20)" rx="2" ry="2" />
<text  x="996.71" y="319.5" ></text>
</g>
<g >
<title>runtime.mallocgc (46 samples, 0.01%)</title><rect x="53.1" y="693" width="0.2" height="15.0" fill="rgb(253,70,34)" rx="2" ry="2" />
<text  x="56.13" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*handshake).start (1,530 samples, 0.47%)</title><rect x="1026.4" y="805" width="5.5" height="15.0" fill="rgb(221,197,38)" rx="2" ry="2" />
<text  x="1029.36" y="815.5" ></text>
</g>
<g >
<title>runtime.assertE2I (53 samples, 0.02%)</title><rect x="186.1" y="661" width="0.2" height="15.0" fill="rgb(250,146,27)" rx="2" ry="2" />
<text  x="189.12" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).getFPRegs (74 samples, 0.02%)</title><rect x="638.3" y="821" width="0.3" height="15.0" fill="rgb(242,178,15)" rx="2" ry="2" />
<text  x="641.29" y="831.5" ></text>
</g>
<g >
<title>runtime.checkTimers (64 samples, 0.02%)</title><rect x="898.0" y="757" width="0.2" height="15.0" fill="rgb(206,151,19)" rx="2" ry="2" />
<text  x="900.96" y="767.5" ></text>
</g>
<g >
<title>runtime.makeslice (169 samples, 0.05%)</title><rect x="729.8" y="693" width="0.6" height="15.0" fill="rgb(232,225,4)" rx="2" ry="2" />
<text  x="732.77" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (55 samples, 0.02%)</title><rect x="701.8" y="677" width="0.2" height="15.0" fill="rgb(251,150,22)" rx="2" ry="2" />
<text  x="704.80" y="687.5" ></text>
</g>
<g >
<title>runtime.convI2I (33 samples, 0.01%)</title><rect x="206.7" y="661" width="0.1" height="15.0" fill="rgb(239,80,47)" rx="2" ry="2" />
<text  x="209.68" y="671.5" ></text>
</g>
<g >
<title>runtime.templateThread (47 samples, 0.01%)</title><rect x="16.8" y="805" width="0.1" height="15.0" fill="rgb(233,77,11)" rx="2" ry="2" />
<text  x="19.75" y="815.5" ></text>
</g>
<g >
<title>runtime.pcdatavalue (232 samples, 0.07%)</title><rect x="1096.2" y="693" width="0.8" height="15.0" fill="rgb(230,155,52)" rx="2" ry="2" />
<text  x="1099.16" y="703.5" ></text>
</g>
<g >
<title>runtime.checkTimers (295 samples, 0.09%)</title><rect x="1057.1" y="757" width="1.1" height="15.0" fill="rgb(219,178,23)" rx="2" ry="2" />
<text  x="1060.14" y="767.5" ></text>
</g>
<g >
<title>runtime.findrunnable (14,169 samples, 4.34%)</title><rect x="98.1" y="645" width="51.3" height="15.0" fill="rgb(254,46,49)" rx="2" ry="2" />
<text  x="101.11" y="655.5" >runti..</text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (277 samples, 0.08%)</title><rect x="765.5" y="517" width="1.0" height="15.0" fill="rgb(213,127,47)" rx="2" ry="2" />
<text  x="768.49" y="527.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendRaw (2,655 samples, 0.81%)</title><rect x="1089.7" y="805" width="9.6" height="15.0" fill="rgb(216,65,16)" rx="2" ry="2" />
<text  x="1092.70" y="815.5" ></text>
</g>
<g >
<title>runtime.mapiternext (96 samples, 0.03%)</title><rect x="737.9" y="661" width="0.3" height="15.0" fill="rgb(239,30,20)" rx="2" ry="2" />
<text  x="740.89" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).enqueuePacketBuffer (278 samples, 0.09%)</title><rect x="181.0" y="501" width="1.0" height="15.0" fill="rgb(243,102,9)" rx="2" ry="2" />
<text  x="184.01" y="511.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (338 samples, 0.10%)</title><rect x="847.4" y="773" width="1.2" height="15.0" fill="rgb(237,114,39)" rx="2" ry="2" />
<text  x="850.42" y="783.5" ></text>
</g>
<g >
<title>__nf_conntrack_find_get (192 samples, 0.06%)</title><rect x="960.4" y="453" width="0.7" height="15.0" fill="rgb(226,196,46)" rx="2" ry="2" />
<text  x="963.42" y="463.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (90 samples, 0.03%)</title><rect x="19.7" y="709" width="0.3" height="15.0" fill="rgb(232,60,18)" rx="2" ry="2" />
<text  x="22.66" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).RecvMsg (33 samples, 0.01%)</title><rect x="173.7" y="757" width="0.2" height="15.0" fill="rgb(241,2,9)" rx="2" ry="2" />
<text  x="176.74" y="767.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (354 samples, 0.11%)</title><rect x="715.6" y="773" width="1.3" height="15.0" fill="rgb(254,56,40)" rx="2" ry="2" />
<text  x="718.62" y="783.5" ></text>
</g>
<g >
<title>runtime.findrunnable (128 samples, 0.04%)</title><rect x="1129.3" y="805" width="0.4" height="15.0" fill="rgb(253,186,41)" rx="2" ry="2" />
<text  x="1132.27" y="815.5" ></text>
</g>
<g >
<title>runtime.mallocgc (88 samples, 0.03%)</title><rect x="711.9" y="789" width="0.3" height="15.0" fill="rgb(235,6,27)" rx="2" ry="2" />
<text  x="714.91" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*syscallTSCReferenceClocks).Cycles (52 samples, 0.02%)</title><rect x="60.3" y="693" width="0.1" height="15.0" fill="rgb(209,188,24)" rx="2" ry="2" />
<text  x="63.26" y="703.5" ></text>
</g>
<g >
<title>sync.(*Pool).Get (190 samples, 0.06%)</title><rect x="70.7" y="709" width="0.7" height="15.0" fill="rgb(250,125,38)" rx="2" ry="2" />
<text  x="73.74" y="719.5" ></text>
</g>
<g >
<title>runtime.write1 (29 samples, 0.01%)</title><rect x="1127.0" y="805" width="0.1" height="15.0" fill="rgb(218,190,8)" rx="2" ry="2" />
<text  x="1130.02" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (48 samples, 0.01%)</title><rect x="776.7" y="661" width="0.1" height="15.0" fill="rgb(213,101,8)" rx="2" ry="2" />
<text  x="779.67" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (26,106 samples, 8.00%)</title><rect x="543.6" y="741" width="94.4" height="15.0" fill="rgb(251,177,11)" rx="2" ry="2" />
<text  x="546.61" y="751.5" >[[kernel.ka..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="1153.4" y="581" width="0.1" height="15.0" fill="rgb(254,142,13)" rx="2" ry="2" />
<text  x="1156.40" y="591.5" ></text>
</g>
<g >
<title>runtime.adjusttimers (461 samples, 0.14%)</title><rect x="1116.7" y="789" width="1.6" height="15.0" fill="rgb(239,196,50)" rx="2" ry="2" />
<text  x="1119.66" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (82 samples, 0.03%)</title><rect x="1028.5" y="421" width="0.3" height="15.0" fill="rgb(212,20,51)" rx="2" ry="2" />
<text  x="1031.48" y="431.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (144 samples, 0.04%)</title><rect x="446.1" y="581" width="0.5" height="15.0" fill="rgb(214,181,9)" rx="2" ry="2" />
<text  x="449.12" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (108 samples, 0.03%)</title><rect x="1085.8" y="677" width="0.4" height="15.0" fill="rgb(224,90,18)" rx="2" ry="2" />
<text  x="1088.79" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (110 samples, 0.03%)</title><rect x="702.5" y="645" width="0.4" height="15.0" fill="rgb(224,56,0)" rx="2" ry="2" />
<text  x="705.46" y="655.5" ></text>
</g>
<g >
<title>runtime.memmove (77 samples, 0.02%)</title><rect x="62.3" y="661" width="0.3" height="15.0" fill="rgb(233,141,48)" rx="2" ry="2" />
<text  x="65.30" y="671.5" ></text>
</g>
<g >
<title>runtime.unlock2 (40 samples, 0.01%)</title><rect x="1064.8" y="741" width="0.2" height="15.0" fill="rgb(238,0,32)" rx="2" ry="2" />
<text  x="1067.84" y="751.5" ></text>
</g>
<g >
<title>runtime.schedule (32 samples, 0.01%)</title><rect x="52.6" y="661" width="0.1" height="15.0" fill="rgb(235,16,9)" rx="2" ry="2" />
<text  x="55.61" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportDemuxer).singleRegisterEndpoint (479 samples, 0.15%)</title><rect x="1022.5" y="789" width="1.8" height="15.0" fill="rgb(214,99,7)" rx="2" ry="2" />
<text  x="1025.54" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.PseudoHeaderChecksum (54 samples, 0.02%)</title><rect x="1075.9" y="693" width="0.2" height="15.0" fill="rgb(222,116,14)" rx="2" ry="2" />
<text  x="1078.89" y="703.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (294 samples, 0.09%)</title><rect x="847.4" y="757" width="1.1" height="15.0" fill="rgb(206,181,5)" rx="2" ry="2" />
<text  x="850.43" y="767.5" ></text>
</g>
<g >
<title>runtime.newobject (30 samples, 0.01%)</title><rect x="895.3" y="837" width="0.1" height="15.0" fill="rgb(241,136,2)" rx="2" ry="2" />
<text  x="898.31" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).init (1,083 samples, 0.33%)</title><rect x="1079.6" y="661" width="3.9" height="15.0" fill="rgb(249,3,12)" rx="2" ry="2" />
<text  x="1082.56" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).entry (81 samples, 0.02%)</title><rect x="1029.3" y="629" width="0.3" height="15.0" fill="rgb(251,1,30)" rx="2" ry="2" />
<text  x="1032.31" y="639.5" ></text>
</g>
<g >
<title>runtime.futex (161 samples, 0.05%)</title><rect x="451.7" y="661" width="0.5" height="15.0" fill="rgb(210,213,38)" rx="2" ry="2" />
<text  x="454.65" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (84 samples, 0.03%)</title><rect x="13.7" y="677" width="0.3" height="15.0" fill="rgb(215,210,48)" rx="2" ry="2" />
<text  x="16.67" y="687.5" ></text>
</g>
<g >
<title>runtime.read (65 samples, 0.02%)</title><rect x="905.3" y="741" width="0.2" height="15.0" fill="rgb(227,31,17)" rx="2" ry="2" />
<text  x="908.30" y="751.5" ></text>
</g>
<g >
<title>runtime.checkTimers (544 samples, 0.17%)</title><rect x="902.3" y="741" width="2.0" height="15.0" fill="rgb(213,152,29)" rx="2" ry="2" />
<text  x="905.28" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="1061.3" y="661" width="0.2" height="15.0" fill="rgb(244,113,20)" rx="2" ry="2" />
<text  x="1064.34" y="671.5" ></text>
</g>
<g >
<title>ovl_write_iter (1,653 samples, 0.51%)</title><rect x="195.4" y="405" width="6.0" height="15.0" fill="rgb(233,47,47)" rx="2" ry="2" />
<text  x="198.43" y="415.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*WeakRef).get (127 samples, 0.04%)</title><rect x="185.9" y="677" width="0.4" height="15.0" fill="rgb(245,206,5)" rx="2" ry="2" />
<text  x="188.86" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*timer).init (164 samples, 0.05%)</title><rect x="1040.6" y="773" width="0.6" height="15.0" fill="rgb(236,144,11)" rx="2" ry="2" />
<text  x="1043.58" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doSyscall (50,585 samples, 15.50%)</title><rect x="38.3" y="837" width="182.9" height="15.0" fill="rgb(237,196,35)" rx="2" ry="2" />
<text  x="41.25" y="847.5" >gvisor.dev/gvisor/pkg/s..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (111 samples, 0.03%)</title><rect x="1060.4" y="613" width="0.4" height="15.0" fill="rgb(251,184,50)" rx="2" ry="2" />
<text  x="1063.40" y="623.5" ></text>
</g>
<g >
<title>runtime.unlock2 (62 samples, 0.02%)</title><rect x="841.3" y="741" width="0.3" height="15.0" fill="rgb(213,68,50)" rx="2" ry="2" />
<text  x="844.35" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="29.4" y="853" width="0.1" height="15.0" fill="rgb(219,170,48)" rx="2" ry="2" />
<text  x="32.38" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborEntry).cancelJobLocked (1,432 samples, 0.44%)</title><rect x="1050.5" y="789" width="5.2" height="15.0" fill="rgb(215,199,12)" rx="2" ry="2" />
<text  x="1053.53" y="799.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (112 samples, 0.03%)</title><rect x="740.1" y="677" width="0.4" height="15.0" fill="rgb(220,76,37)" rx="2" ry="2" />
<text  x="743.10" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).SleepFinish (779 samples, 0.24%)</title><rect x="80.1" y="709" width="2.9" height="15.0" fill="rgb(250,178,15)" rx="2" ry="2" />
<text  x="83.14" y="719.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (34 samples, 0.01%)</title><rect x="1076.5" y="661" width="0.2" height="15.0" fill="rgb(239,45,51)" rx="2" ry="2" />
<text  x="1079.54" y="671.5" ></text>
</g>
<g >
<title>runtime.(*mspan).sweep (253 samples, 0.08%)</title><rect x="1110.1" y="837" width="0.9" height="15.0" fill="rgb(247,46,25)" rx="2" ry="2" />
<text  x="1113.08" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="1173.1" y="805" width="0.1" height="15.0" fill="rgb(214,172,49)" rx="2" ry="2" />
<text  x="1176.09" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleSegments (41 samples, 0.01%)</title><rect x="1107.1" y="853" width="0.1" height="15.0" fill="rgb(225,179,43)" rx="2" ry="2" />
<text  x="1110.09" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).Release (75 samples, 0.02%)</title><rect x="1100.7" y="821" width="0.2" height="15.0" fill="rgb(214,50,11)" rx="2" ry="2" />
<text  x="1103.66" y="831.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (40 samples, 0.01%)</title><rect x="183.2" y="629" width="0.1" height="15.0" fill="rgb(229,184,16)" rx="2" ry="2" />
<text  x="186.18" y="639.5" ></text>
</g>
<g >
<title>runtime.(*spanSet).push (35 samples, 0.01%)</title><rect x="767.2" y="533" width="0.1" height="15.0" fill="rgb(253,1,5)" rx="2" ry="2" />
<text  x="770.17" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/usermem.(*IOSequenceReadWriter).Read (278 samples, 0.09%)</title><rect x="205.7" y="645" width="1.0" height="15.0" fill="rgb(220,110,41)" rx="2" ry="2" />
<text  x="208.65" y="655.5" ></text>
</g>
<g >
<title>runtime.runqsteal (97 samples, 0.03%)</title><rect x="22.3" y="805" width="0.3" height="15.0" fill="rgb(236,175,8)" rx="2" ry="2" />
<text  x="25.27" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacketBuffer (241 samples, 0.07%)</title><rect x="1073.6" y="629" width="0.8" height="15.0" fill="rgb(223,201,12)" rx="2" ry="2" />
<text  x="1076.58" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="637.9" y="597" width="0.1" height="15.0" fill="rgb(228,201,41)" rx="2" ry="2" />
<text  x="640.87" y="607.5" ></text>
</g>
<g >
<title>runtime.newobject (35 samples, 0.01%)</title><rect x="58.7" y="741" width="0.1" height="15.0" fill="rgb(241,32,46)" rx="2" ry="2" />
<text  x="61.67" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*endpoint).Capabilities (45 samples, 0.01%)</title><rect x="723.0" y="709" width="0.2" height="15.0" fill="rgb(236,9,7)" rx="2" ry="2" />
<text  x="726.02" y="719.5" ></text>
</g>
<g >
<title>runtime.entersyscallblock (186 samples, 0.06%)</title><rect x="822.5" y="805" width="0.7" height="15.0" fill="rgb(224,126,21)" rx="2" ry="2" />
<text  x="825.51" y="815.5" ></text>
</g>
<g >
<title>runtime.mapiterinit (56 samples, 0.02%)</title><rect x="1037.2" y="677" width="0.2" height="15.0" fill="rgb(229,206,27)" rx="2" ry="2" />
<text  x="1040.19" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (42 samples, 0.01%)</title><rect x="1027.9" y="565" width="0.1" height="15.0" fill="rgb(248,54,0)" rx="2" ry="2" />
<text  x="1030.86" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,031 samples, 0.32%)</title><rect x="123.6" y="565" width="3.8" height="15.0" fill="rgb(213,211,52)" rx="2" ry="2" />
<text  x="126.64" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverNetworkPacket (21,238 samples, 6.51%)</title><rect x="719.1" y="773" width="76.8" height="15.0" fill="rgb(207,189,50)" rx="2" ry="2" />
<text  x="722.14" y="783.5" >gvisor.d..</text>
</g>
<g >
<title>runtime.park_m (2,733 samples, 0.84%)</title><rect x="1056.9" y="789" width="9.9" height="15.0" fill="rgb(246,68,46)" rx="2" ry="2" />
<text  x="1059.92" y="799.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (134 samples, 0.04%)</title><rect x="711.1" y="725" width="0.5" height="15.0" fill="rgb(240,87,3)" rx="2" ry="2" />
<text  x="714.14" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (314 samples, 0.10%)</title><rect x="920.7" y="597" width="1.2" height="15.0" fill="rgb(243,15,1)" rx="2" ry="2" />
<text  x="923.74" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Accept (41 samples, 0.01%)</title><rect x="57.8" y="741" width="0.2" height="15.0" fill="rgb(227,178,8)" rx="2" ry="2" />
<text  x="60.85" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*segmentList).Remove (40 samples, 0.01%)</title><rect x="1089.3" y="821" width="0.1" height="15.0" fill="rgb(232,129,33)" rx="2" ry="2" />
<text  x="1092.28" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (12,135 samples, 3.72%)</title><rect x="851.0" y="805" width="43.9" height="15.0" fill="rgb(251,20,4)" rx="2" ry="2" />
<text  x="854.00" y="815.5" >[[ke..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).MaxHeaderLength (34 samples, 0.01%)</title><rect x="209.1" y="549" width="0.1" height="15.0" fill="rgb(250,105,18)" rx="2" ry="2" />
<text  x="212.08" y="559.5" ></text>
</g>
<g >
<title>runtime.mallocgc (44 samples, 0.01%)</title><rect x="1050.2" y="757" width="0.2" height="15.0" fill="rgb(214,128,34)" rx="2" ry="2" />
<text  x="1053.23" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).get (89 samples, 0.03%)</title><rect x="213.7" y="485" width="0.3" height="15.0" fill="rgb(242,17,0)" rx="2" ry="2" />
<text  x="216.67" y="495.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (813 samples, 0.25%)</title><rect x="145.5" y="405" width="2.9" height="15.0" fill="rgb(221,152,27)" rx="2" ry="2" />
<text  x="148.47" y="415.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (33 samples, 0.01%)</title><rect x="1061.4" y="645" width="0.1" height="15.0" fill="rgb(227,183,27)" rx="2" ry="2" />
<text  x="1064.38" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendData (78 samples, 0.02%)</title><rect x="1088.1" y="789" width="0.2" height="15.0" fill="rgb(248,206,33)" rx="2" ry="2" />
<text  x="1091.05" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).DeliverNetworkPacket (21,816 samples, 6.69%)</title><rect x="718.1" y="821" width="78.9" height="15.0" fill="rgb(233,187,41)" rx="2" ry="2" />
<text  x="721.11" y="831.5" >gvisor.de..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/fsutil.(*inodeReadWriter).WriteFromBlocks (47 samples, 0.01%)</title><rect x="21.6" y="677" width="0.1" height="15.0" fill="rgb(242,162,51)" rx="2" ry="2" />
<text  x="24.57" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).AfterFunc (758 samples, 0.23%)</title><rect x="1047.4" y="773" width="2.8" height="15.0" fill="rgb(243,214,18)" rx="2" ry="2" />
<text  x="1050.42" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*recvMMsgDispatcher).dispatch (51 samples, 0.02%)</title><rect x="895.6" y="853" width="0.1" height="15.0" fill="rgb(214,160,51)" rx="2" ry="2" />
<text  x="898.56" y="863.5" ></text>
</g>
<g >
<title>runtime.read (95 samples, 0.03%)</title><rect x="1120.0" y="805" width="0.4" height="15.0" fill="rgb(251,132,23)" rx="2" ry="2" />
<text  x="1123.04" y="815.5" ></text>
</g>
<g >
<title>runtime.chanrecv (62 samples, 0.02%)</title><rect x="54.7" y="693" width="0.2" height="15.0" fill="rgb(222,141,25)" rx="2" ry="2" />
<text  x="57.68" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.ParseSynOptions (39 samples, 0.01%)</title><rect x="1044.4" y="821" width="0.1" height="15.0" fill="rgb(238,125,47)" rx="2" ry="2" />
<text  x="1047.38" y="831.5" ></text>
</g>
<g >
<title>nfnetlink_has_listeners (36 samples, 0.01%)</title><rect x="993.6" y="261" width="0.1" height="15.0" fill="rgb(251,5,44)" rx="2" ry="2" />
<text  x="996.58" y="271.5" ></text>
</g>
<g >
<title>br_validate_ipv4.isra.30 (39 samples, 0.01%)</title><rect x="995.1" y="485" width="0.2" height="15.0" fill="rgb(252,216,48)" rx="2" ry="2" />
<text  x="998.14" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).WritePacket (910 samples, 0.28%)</title><rect x="210.1" y="469" width="3.3" height="15.0" fill="rgb(208,182,0)" rx="2" ry="2" />
<text  x="213.07" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (63 samples, 0.02%)</title><rect x="1091.6" y="581" width="0.2" height="15.0" fill="rgb(232,218,14)" rx="2" ry="2" />
<text  x="1094.61" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).Enabled (84 samples, 0.03%)</title><rect x="786.5" y="677" width="0.3" height="15.0" fill="rgb(214,167,15)" rx="2" ry="2" />
<text  x="789.47" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*pmaSet).Find (70 samples, 0.02%)</title><rect x="62.6" y="677" width="0.3" height="15.0" fill="rgb(242,169,28)" rx="2" ry="2" />
<text  x="65.64" y="687.5" ></text>
</g>
<g >
<title>runtime.mcall (516 samples, 0.16%)</title><rect x="1107.5" y="837" width="1.9" height="15.0" fill="rgb(234,143,9)" rx="2" ry="2" />
<text  x="1110.49" y="847.5" ></text>
</g>
<g >
<title>ip_sabotage_in (37 samples, 0.01%)</title><rect x="959.2" y="469" width="0.2" height="15.0" fill="rgb(249,5,3)" rx="2" ry="2" />
<text  x="962.23" y="479.5" ></text>
</g>
<g >
<title>runtime.readvarint (28 samples, 0.01%)</title><rect x="1097.7" y="677" width="0.1" height="15.0" fill="rgb(216,8,24)" rx="2" ry="2" />
<text  x="1100.66" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.makeRouteInner (132 samples, 0.04%)</title><rect x="1033.7" y="725" width="0.5" height="15.0" fill="rgb(213,58,49)" rx="2" ry="2" />
<text  x="1036.75" y="735.5" ></text>
</g>
<g >
<title>runtime.selunlock (143 samples, 0.04%)</title><rect x="160.8" y="693" width="0.5" height="15.0" fill="rgb(226,41,18)" rx="2" ry="2" />
<text  x="163.77" y="703.5" ></text>
</g>
<g >
<title>runtime.futex (55 samples, 0.02%)</title><rect x="16.9" y="853" width="0.2" height="15.0" fill="rgb(229,67,48)" rx="2" ry="2" />
<text  x="19.92" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (274 samples, 0.08%)</title><rect x="700.6" y="725" width="1.0" height="15.0" fill="rgb(244,161,28)" rx="2" ry="2" />
<text  x="703.60" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (65 samples, 0.02%)</title><rect x="1189.8" y="661" width="0.2" height="15.0" fill="rgb(217,61,17)" rx="2" ry="2" />
<text  x="1192.76" y="671.5" ></text>
</g>
<g >
<title>runtime.runqgrab (103 samples, 0.03%)</title><rect x="118.4" y="629" width="0.3" height="15.0" fill="rgb(205,102,33)" rx="2" ry="2" />
<text  x="121.36" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (77 samples, 0.02%)</title><rect x="1007.4" y="629" width="0.3" height="15.0" fill="rgb(254,4,3)" rx="2" ry="2" />
<text  x="1010.38" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress.func1 (134 samples, 0.04%)</title><rect x="1036.3" y="677" width="0.5" height="15.0" fill="rgb(240,224,46)" rx="2" ry="2" />
<text  x="1039.33" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).AcquireAssignedAddressOrMatching (36 samples, 0.01%)</title><rect x="1032.9" y="693" width="0.2" height="15.0" fill="rgb(220,131,1)" rx="2" ry="2" />
<text  x="1035.94" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (50 samples, 0.02%)</title><rect x="23.5" y="693" width="0.2" height="15.0" fill="rgb(228,28,47)" rx="2" ry="2" />
<text  x="26.52" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (55 samples, 0.02%)</title><rect x="117.9" y="533" width="0.2" height="15.0" fill="rgb(242,113,4)" rx="2" ry="2" />
<text  x="120.89" y="543.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (220 samples, 0.07%)</title><rect x="1127.8" y="709" width="0.8" height="15.0" fill="rgb(236,179,46)" rx="2" ry="2" />
<text  x="1130.77" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/usermem.CopyInVec (234 samples, 0.07%)</title><rect x="205.8" y="629" width="0.9" height="15.0" fill="rgb(249,205,36)" rx="2" ry="2" />
<text  x="208.81" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (62 samples, 0.02%)</title><rect x="1120.2" y="773" width="0.2" height="15.0" fill="rgb(226,210,22)" rx="2" ry="2" />
<text  x="1123.16" y="783.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).uncacheSpan (49 samples, 0.02%)</title><rect x="767.1" y="549" width="0.2" height="15.0" fill="rgb(250,221,46)" rx="2" ry="2" />
<text  x="770.12" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).CheckPermission (286 samples, 0.09%)</title><rect x="188.0" y="709" width="1.0" height="15.0" fill="rgb(254,109,33)" rx="2" ry="2" />
<text  x="190.96" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*multiPortEndpoint).singleRegisterEndpoint (52 samples, 0.02%)</title><rect x="1022.8" y="757" width="0.2" height="15.0" fill="rgb(245,217,2)" rx="2" ry="2" />
<text  x="1025.77" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (111 samples, 0.03%)</title><rect x="157.4" y="421" width="0.4" height="15.0" fill="rgb(223,200,29)" rx="2" ry="2" />
<text  x="160.42" y="431.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/binary.marshal (393 samples, 0.12%)</title><rect x="924.2" y="789" width="1.4" height="15.0" fill="rgb(234,67,6)" rx="2" ry="2" />
<text  x="927.23" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="23.1" y="597" width="0.2" height="15.0" fill="rgb(219,94,39)" rx="2" ry="2" />
<text  x="26.13" y="607.5" ></text>
</g>
<g >
<title>runtime.mallocgc (61 samples, 0.02%)</title><rect x="1029.7" y="629" width="0.2" height="15.0" fill="rgb(229,46,1)" rx="2" ry="2" />
<text  x="1032.71" y="639.5" ></text>
</g>
<g >
<title>nf_nat_inet_fn (48 samples, 0.01%)</title><rect x="964.8" y="453" width="0.1" height="15.0" fill="rgb(243,54,43)" rx="2" ry="2" />
<text  x="967.78" y="463.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,505 samples, 0.46%)</title><rect x="816.7" y="645" width="5.5" height="15.0" fill="rgb(210,92,4)" rx="2" ry="2" />
<text  x="819.72" y="655.5" ></text>
</g>
<g >
<title>runtime.growslice (34 samples, 0.01%)</title><rect x="1022.8" y="741" width="0.2" height="15.0" fill="rgb(215,177,13)" rx="2" ry="2" />
<text  x="1025.83" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/fsutil.NewInodeSimpleAttributes (123 samples, 0.04%)</title><rect x="51.8" y="693" width="0.5" height="15.0" fill="rgb(239,217,38)" rx="2" ry="2" />
<text  x="54.81" y="703.5" ></text>
</g>
<g >
<title>runtime.sysmon (862 samples, 0.26%)</title><rect x="12.7" y="773" width="3.2" height="15.0" fill="rgb(226,171,17)" rx="2" ry="2" />
<text  x="15.75" y="783.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (77 samples, 0.02%)</title><rect x="762.6" y="597" width="0.3" height="15.0" fill="rgb(253,115,8)" rx="2" ry="2" />
<text  x="765.61" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).WritePacket (392 samples, 0.12%)</title><rect x="180.8" y="565" width="1.4" height="15.0" fill="rgb(232,40,38)" rx="2" ry="2" />
<text  x="183.82" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).getAddressOrCreateTemp (2,967 samples, 0.91%)</title><rect x="782.0" y="741" width="10.8" height="15.0" fill="rgb(245,144,36)" rx="2" ry="2" />
<text  x="785.05" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (37 samples, 0.01%)</title><rect x="778.9" y="709" width="0.2" height="15.0" fill="rgb(238,92,53)" rx="2" ry="2" />
<text  x="781.95" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/packetsocket.(*endpoint).WritePacket (887 samples, 0.27%)</title><rect x="210.1" y="453" width="3.2" height="15.0" fill="rgb(243,221,24)" rx="2" ry="2" />
<text  x="213.12" y="463.5" ></text>
</g>
<g >
<title>runtime.save (47 samples, 0.01%)</title><rect x="448.3" y="741" width="0.2" height="15.0" fill="rgb(236,100,33)" rx="2" ry="2" />
<text  x="451.33" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Accept4 (67 samples, 0.02%)</title><rect x="218.8" y="789" width="0.2" height="15.0" fill="rgb(220,116,48)" rx="2" ry="2" />
<text  x="221.79" y="799.5" ></text>
</g>
<g >
<title>ipv4_conntrack_in (54 samples, 0.02%)</title><rect x="959.7" y="469" width="0.2" height="15.0" fill="rgb(224,35,33)" rx="2" ry="2" />
<text  x="962.67" y="479.5" ></text>
</g>
<g >
<title>runtime.notesleep (919 samples, 0.28%)</title><rect x="1123.5" y="789" width="3.3" height="15.0" fill="rgb(205,197,14)" rx="2" ry="2" />
<text  x="1126.49" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*timer).cleanup (64 samples, 0.02%)</title><rect x="1069.9" y="837" width="0.2" height="15.0" fill="rgb(240,110,35)" rx="2" ry="2" />
<text  x="1072.86" y="847.5" ></text>
</g>
<g >
<title>br_nf_pre_routing_finish (8,301 samples, 2.54%)</title><rect x="965.1" y="485" width="30.0" height="15.0" fill="rgb(226,125,37)" rx="2" ry="2" />
<text  x="968.12" y="495.5" >br..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (672 samples, 0.21%)</title><rect x="954.1" y="453" width="2.4" height="15.0" fill="rgb(213,148,13)" rx="2" ry="2" />
<text  x="957.11" y="463.5" ></text>
</g>
<g >
<title>runtime.makechan (67 samples, 0.02%)</title><rect x="47.7" y="693" width="0.3" height="15.0" fill="rgb(252,225,5)" rx="2" ry="2" />
<text  x="50.75" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.newEndpoint (982 samples, 0.30%)</title><rect x="1039.4" y="789" width="3.6" height="15.0" fill="rgb(218,38,18)" rx="2" ry="2" />
<text  x="1042.44" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (69 samples, 0.02%)</title><rect x="156.0" y="389" width="0.3" height="15.0" fill="rgb(245,65,12)" rx="2" ry="2" />
<text  x="159.04" y="399.5" ></text>
</g>
<g >
<title>runtime.makeslice (28 samples, 0.01%)</title><rect x="1033.5" y="709" width="0.1" height="15.0" fill="rgb(214,193,2)" rx="2" ry="2" />
<text  x="1036.51" y="719.5" ></text>
</g>
<g >
<title>__ext4_journal_stop (51 samples, 0.02%)</title><rect x="200.9" y="293" width="0.2" height="15.0" fill="rgb(215,16,10)" rx="2" ry="2" />
<text  x="203.88" y="303.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).commitRead (42 samples, 0.01%)</title><rect x="177.1" y="693" width="0.2" height="15.0" fill="rgb(210,146,29)" rx="2" ry="2" />
<text  x="180.10" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*AtomicRefCount).DecRefWithDestructor (237 samples, 0.07%)</title><rect x="63.9" y="693" width="0.8" height="15.0" fill="rgb(245,89,13)" rx="2" ry="2" />
<text  x="66.85" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (48 samples, 0.01%)</title><rect x="1005.9" y="693" width="0.2" height="15.0" fill="rgb(239,159,18)" rx="2" ry="2" />
<text  x="1008.91" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/syserror.TranslateError (32 samples, 0.01%)</title><rect x="220.0" y="789" width="0.2" height="15.0" fill="rgb(242,5,31)" rx="2" ry="2" />
<text  x="223.04" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (211 samples, 0.06%)</title><rect x="1186.9" y="693" width="0.7" height="15.0" fill="rgb(209,154,17)" rx="2" ry="2" />
<text  x="1189.85" y="703.5" ></text>
</g>
<g >
<title>runtime.runqgrab (1,326 samples, 0.41%)</title><rect x="906.0" y="725" width="4.8" height="15.0" fill="rgb(216,15,10)" rx="2" ry="2" />
<text  x="908.99" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.EpollWait (27,825 samples, 8.53%)</title><rect x="72.9" y="773" width="100.6" height="15.0" fill="rgb(210,131,19)" rx="2" ry="2" />
<text  x="75.88" y="783.5" >gvisor.dev/g..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (550 samples, 0.17%)</title><rect x="908.8" y="613" width="2.0" height="15.0" fill="rgb(216,169,9)" rx="2" ry="2" />
<text  x="911.79" y="623.5" ></text>
</g>
<g >
<title>runtime.mapiternext (40 samples, 0.01%)</title><rect x="1037.4" y="677" width="0.1" height="15.0" fill="rgb(223,66,49)" rx="2" ry="2" />
<text  x="1040.40" y="687.5" ></text>
</g>
<g >
<title>runtime.runqsteal (50 samples, 0.02%)</title><rect x="1108.3" y="773" width="0.2" height="15.0" fill="rgb(212,219,54)" rx="2" ry="2" />
<text  x="1111.29" y="783.5" ></text>
</g>
<g >
<title>runtime.casgstatus (32 samples, 0.01%)</title><rect x="898.2" y="741" width="0.1" height="15.0" fill="rgb(246,124,29)" rx="2" ry="2" />
<text  x="901.23" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doSyscall (35 samples, 0.01%)</title><rect x="34.6" y="853" width="0.2" height="15.0" fill="rgb(231,136,15)" rx="2" ry="2" />
<text  x="37.65" y="863.5" ></text>
</g>
<g >
<title>runtime.mapassign (61 samples, 0.02%)</title><rect x="1031.9" y="789" width="0.2" height="15.0" fill="rgb(245,227,5)" rx="2" ry="2" />
<text  x="1034.92" y="799.5" ></text>
</g>
<g >
<title>runtime.stopm (2,189 samples, 0.67%)</title><rect x="910.9" y="741" width="7.9" height="15.0" fill="rgb(246,162,23)" rx="2" ry="2" />
<text  x="913.87" y="751.5" ></text>
</g>
<g >
<title>runtime.copystack (1,826 samples, 0.56%)</title><rect x="1147.4" y="757" width="6.6" height="15.0" fill="rgb(225,220,15)" rx="2" ry="2" />
<text  x="1150.39" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (275 samples, 0.08%)</title><rect x="703.9" y="725" width="1.0" height="15.0" fill="rgb(248,204,8)" rx="2" ry="2" />
<text  x="706.88" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*protocol).QueuePacket (92 samples, 0.03%)</title><rect x="19.7" y="757" width="0.3" height="15.0" fill="rgb(212,150,47)" rx="2" ry="2" />
<text  x="22.66" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (32 samples, 0.01%)</title><rect x="163.3" y="693" width="0.1" height="15.0" fill="rgb(216,26,39)" rx="2" ry="2" />
<text  x="166.29" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (154 samples, 0.05%)</title><rect x="1060.2" y="661" width="0.6" height="15.0" fill="rgb(218,12,13)" rx="2" ry="2" />
<text  x="1063.25" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBufferList).Remove (61 samples, 0.02%)</title><rect x="999.1" y="837" width="0.2" height="15.0" fill="rgb(254,160,39)" rx="2" ry="2" />
<text  x="1002.11" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (41 samples, 0.01%)</title><rect x="1075.7" y="645" width="0.1" height="15.0" fill="rgb(251,134,28)" rx="2" ry="2" />
<text  x="1078.66" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (156 samples, 0.05%)</title><rect x="1060.2" y="677" width="0.6" height="15.0" fill="rgb(208,112,25)" rx="2" ry="2" />
<text  x="1063.24" y="687.5" ></text>
</g>
<g >
<title>nft_do_chain (138 samples, 0.04%)</title><rect x="994.1" y="293" width="0.5" height="15.0" fill="rgb(230,181,34)" rx="2" ry="2" />
<text  x="997.08" y="303.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (167 samples, 0.05%)</title><rect x="126.8" y="421" width="0.6" height="15.0" fill="rgb(231,197,29)" rx="2" ry="2" />
<text  x="129.77" y="431.5" ></text>
</g>
<g >
<title>runtime.chanrecv1 (165 samples, 0.05%)</title><rect x="451.6" y="773" width="0.6" height="15.0" fill="rgb(245,134,35)" rx="2" ry="2" />
<text  x="454.64" y="783.5" ></text>
</g>
<g >
<title>runtime.startm (51 samples, 0.02%)</title><rect x="1109.1" y="757" width="0.2" height="15.0" fill="rgb(238,44,46)" rx="2" ry="2" />
<text  x="1112.12" y="767.5" ></text>
</g>
<g >
<title>runtime.mapaccess1_fast64 (64 samples, 0.02%)</title><rect x="639.9" y="821" width="0.2" height="15.0" fill="rgb(220,80,29)" rx="2" ry="2" />
<text  x="642.90" y="831.5" ></text>
</g>
<g >
<title>runtime.gentraceback (863 samples, 0.26%)</title><rect x="1094.7" y="741" width="3.1" height="15.0" fill="rgb(208,118,42)" rx="2" ry="2" />
<text  x="1097.72" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).drop (107 samples, 0.03%)</title><rect x="69.0" y="741" width="0.4" height="15.0" fill="rgb(233,19,1)" rx="2" ry="2" />
<text  x="72.01" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (13,350 samples, 4.09%)</title><rect x="948.4" y="597" width="48.3" height="15.0" fill="rgb(243,191,28)" rx="2" ry="2" />
<text  x="951.43" y="607.5" >[[ke..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*TcpipTimer).Stop (429 samples, 0.13%)</title><rect x="1084.8" y="709" width="1.5" height="15.0" fill="rgb(248,79,34)" rx="2" ry="2" />
<text  x="1087.78" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).enqueuePacketBuffer (462 samples, 0.14%)</title><rect x="1091.1" y="693" width="1.7" height="15.0" fill="rgb(247,127,42)" rx="2" ry="2" />
<text  x="1094.10" y="703.5" ></text>
</g>
<g >
<title>runtime.mapdelete_fast64 (49 samples, 0.02%)</title><rect x="64.4" y="629" width="0.2" height="15.0" fill="rgb(212,66,24)" rx="2" ry="2" />
<text  x="67.43" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (45 samples, 0.01%)</title><rect x="766.3" y="437" width="0.2" height="15.0" fill="rgb(206,72,28)" rx="2" ry="2" />
<text  x="769.33" y="447.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (76 samples, 0.02%)</title><rect x="13.7" y="645" width="0.3" height="15.0" fill="rgb(253,29,27)" rx="2" ry="2" />
<text  x="16.70" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (111 samples, 0.03%)</title><rect x="196.6" y="229" width="0.4" height="15.0" fill="rgb(226,71,43)" rx="2" ry="2" />
<text  x="199.58" y="239.5" ></text>
</g>
<g >
<title>runtime.mallocgc (105 samples, 0.03%)</title><rect x="1020.0" y="821" width="0.4" height="15.0" fill="rgb(233,118,54)" rx="2" ry="2" />
<text  x="1022.99" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1051.3" y="549" width="0.1" height="15.0" fill="rgb(245,129,15)" rx="2" ry="2" />
<text  x="1054.25" y="559.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (43 samples, 0.01%)</title><rect x="1012.8" y="677" width="0.2" height="15.0" fill="rgb(229,141,16)" rx="2" ry="2" />
<text  x="1015.83" y="687.5" ></text>
</g>
<g >
<title>runtime.read (125 samples, 0.04%)</title><rect x="117.6" y="629" width="0.5" height="15.0" fill="rgb(214,169,22)" rx="2" ry="2" />
<text  x="120.63" y="639.5" ></text>
</g>
<g >
<title>runtime.mcall (5,058 samples, 1.55%)</title><rect x="1111.7" y="869" width="18.3" height="15.0" fill="rgb(228,45,12)" rx="2" ry="2" />
<text  x="1114.70" y="879.5" ></text>
</g>
<g >
<title>runtime.lock2 (29 samples, 0.01%)</title><rect x="844.9" y="757" width="0.1" height="15.0" fill="rgb(245,12,16)" rx="2" ry="2" />
<text  x="847.87" y="767.5" ></text>
</g>
<g >
<title>runtime.casgstatus (41 samples, 0.01%)</title><rect x="1114.0" y="805" width="0.2" height="15.0" fill="rgb(224,133,28)" rx="2" ry="2" />
<text  x="1117.02" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (140 samples, 0.04%)</title><rect x="990.3" y="85" width="0.5" height="15.0" fill="rgb(231,127,53)" rx="2" ry="2" />
<text  x="993.29" y="95.5" ></text>
</g>
<g >
<title>runtime.mallocgc (73 samples, 0.02%)</title><rect x="735.3" y="613" width="0.3" height="15.0" fill="rgb(218,185,1)" rx="2" ry="2" />
<text  x="738.31" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isResolutionRequiredRLocked (35 samples, 0.01%)</title><rect x="1074.5" y="613" width="0.2" height="15.0" fill="rgb(236,186,47)" rx="2" ry="2" />
<text  x="1077.55" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).TryIncRef (83 samples, 0.03%)</title><rect x="186.3" y="677" width="0.3" height="15.0" fill="rgb(207,5,25)" rx="2" ry="2" />
<text  x="189.33" y="687.5" ></text>
</g>
<g >
<title>runtime.funcname (107 samples, 0.03%)</title><rect x="1112.8" y="821" width="0.4" height="15.0" fill="rgb(208,168,14)" rx="2" ry="2" />
<text  x="1115.82" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendSegmentFromView (28 samples, 0.01%)</title><rect x="1107.1" y="837" width="0.1" height="15.0" fill="rgb(218,93,1)" rx="2" ry="2" />
<text  x="1110.13" y="847.5" ></text>
</g>
<g >
<title>runtime.mallocgc (72 samples, 0.02%)</title><rect x="1068.2" y="805" width="0.2" height="15.0" fill="rgb(248,131,35)" rx="2" ry="2" />
<text  x="1071.18" y="815.5" ></text>
</g>
<g >
<title>runtime.ready (31 samples, 0.01%)</title><rect x="1085.3" y="629" width="0.1" height="15.0" fill="rgb(229,54,22)" rx="2" ry="2" />
<text  x="1088.31" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (57 samples, 0.02%)</title><rect x="1028.6" y="373" width="0.2" height="15.0" fill="rgb(213,156,42)" rx="2" ry="2" />
<text  x="1031.57" y="383.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (245 samples, 0.08%)</title><rect x="31.5" y="853" width="0.9" height="15.0" fill="rgb(219,110,22)" rx="2" ry="2" />
<text  x="34.52" y="863.5" ></text>
</g>
<g >
<title>runtime.entersyscallblock_handoff (5,075 samples, 1.56%)</title><rect x="823.5" y="789" width="18.4" height="15.0" fill="rgb(208,1,21)" rx="2" ry="2" />
<text  x="826.52" y="799.5" ></text>
</g>
<g >
<title>runtime.mallocgc (28 samples, 0.01%)</title><rect x="47.9" y="677" width="0.1" height="15.0" fill="rgb(233,115,51)" rx="2" ry="2" />
<text  x="50.89" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (109 samples, 0.03%)</title><rect x="157.4" y="405" width="0.4" height="15.0" fill="rgb(212,182,48)" rx="2" ry="2" />
<text  x="160.43" y="415.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (29 samples, 0.01%)</title><rect x="1120.3" y="709" width="0.1" height="15.0" fill="rgb(245,33,5)" rx="2" ry="2" />
<text  x="1123.28" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (58 samples, 0.02%)</title><rect x="1007.4" y="581" width="0.3" height="15.0" fill="rgb(254,129,49)" rx="2" ry="2" />
<text  x="1010.45" y="591.5" ></text>
</g>
<g >
<title>syscall.Syscall6 (282 samples, 0.09%)</title><rect x="703.9" y="773" width="1.0" height="15.0" fill="rgb(224,98,33)" rx="2" ry="2" />
<text  x="706.86" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).isValidForOutgoing (61 samples, 0.02%)</title><rect x="214.6" y="533" width="0.2" height="15.0" fill="rgb(219,47,41)" rx="2" ry="2" />
<text  x="217.62" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (57 samples, 0.02%)</title><rect x="784.4" y="661" width="0.2" height="15.0" fill="rgb(235,89,8)" rx="2" ry="2" />
<text  x="787.41" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,583 samples, 0.49%)</title><rect x="950.8" y="533" width="5.7" height="15.0" fill="rgb(210,42,44)" rx="2" ry="2" />
<text  x="953.82" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.NowFromContext (99 samples, 0.03%)</title><rect x="51.9" y="677" width="0.3" height="15.0" fill="rgb(215,23,47)" rx="2" ry="2" />
<text  x="54.86" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (220 samples, 0.07%)</title><rect x="1130.1" y="773" width="0.8" height="15.0" fill="rgb(209,177,5)" rx="2" ry="2" />
<text  x="1133.11" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (36 samples, 0.01%)</title><rect x="776.0" y="693" width="0.1" height="15.0" fill="rgb(222,166,43)" rx="2" ry="2" />
<text  x="778.97" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (47 samples, 0.01%)</title><rect x="1061.3" y="693" width="0.2" height="15.0" fill="rgb(252,1,26)" rx="2" ry="2" />
<text  x="1064.33" y="703.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (37 samples, 0.01%)</title><rect x="171.7" y="661" width="0.1" height="15.0" fill="rgb(241,79,21)" rx="2" ry="2" />
<text  x="174.68" y="671.5" ></text>
</g>
<g >
<title>runtime.findfunc (69 samples, 0.02%)</title><rect x="1054.4" y="709" width="0.2" height="15.0" fill="rgb(230,160,35)" rx="2" ry="2" />
<text  x="1057.36" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (533 samples, 0.16%)</title><rect x="1124.9" y="693" width="1.9" height="15.0" fill="rgb(206,204,23)" rx="2" ry="2" />
<text  x="1127.87" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (246 samples, 0.08%)</title><rect x="921.0" y="565" width="0.9" height="15.0" fill="rgb(210,17,21)" rx="2" ry="2" />
<text  x="923.98" y="575.5" ></text>
</g>
<g >
<title>runtime.newobject (55 samples, 0.02%)</title><rect x="48.0" y="693" width="0.2" height="15.0" fill="rgb(212,32,23)" rx="2" ry="2" />
<text  x="51.00" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.AddressWithPrefix.Subnet (464 samples, 0.14%)</title><rect x="729.4" y="709" width="1.7" height="15.0" fill="rgb(251,215,29)" rx="2" ry="2" />
<text  x="732.41" y="719.5" ></text>
</g>
<g >
<title>runtime.ifaceeq (42 samples, 0.01%)</title><rect x="58.5" y="741" width="0.2" height="15.0" fill="rgb(253,135,18)" rx="2" ry="2" />
<text  x="61.51" y="751.5" ></text>
</g>
<g >
<title>runtime.(*mheap).freeManual (34 samples, 0.01%)</title><rect x="1111.1" y="805" width="0.1" height="15.0" fill="rgb(211,174,35)" rx="2" ry="2" />
<text  x="1114.07" y="815.5" ></text>
</g>
<g >
<title>runtime.makeslice (100 samples, 0.03%)</title><rect x="184.4" y="709" width="0.4" height="15.0" fill="rgb(235,161,33)" rx="2" ry="2" />
<text  x="187.44" y="719.5" ></text>
</g>
<g >
<title>runtime.stopm (29 samples, 0.01%)</title><rect x="696.9" y="773" width="0.1" height="15.0" fill="rgb(222,207,6)" rx="2" ry="2" />
<text  x="699.85" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (29 samples, 0.01%)</title><rect x="1099.1" y="741" width="0.1" height="15.0" fill="rgb(229,203,9)" rx="2" ry="2" />
<text  x="1102.13" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.buildTCPHdr (55 samples, 0.02%)</title><rect x="182.2" y="565" width="0.2" height="15.0" fill="rgb(236,36,36)" rx="2" ry="2" />
<text  x="185.24" y="575.5" ></text>
</g>
<g >
<title>runtime.mcall (1,207 samples, 0.37%)</title><rect x="698.7" y="837" width="4.4" height="15.0" fill="rgb(254,31,1)" rx="2" ry="2" />
<text  x="701.73" y="847.5" ></text>
</g>
<g >
<title>runtime.pcvalue (603 samples, 0.18%)</title><rect x="1139.9" y="693" width="2.2" height="15.0" fill="rgb(211,136,27)" rx="2" ry="2" />
<text  x="1142.93" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (45,097 samples, 13.82%)</title><rect x="283.6" y="677" width="163.1" height="15.0" fill="rgb(210,98,29)" rx="2" ry="2" />
<text  x="286.57" y="687.5" >[[kernel.kallsyms]]</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).accountTaskGoroutineLeave (33 samples, 0.01%)</title><rect x="34.2" y="853" width="0.1" height="15.0" fill="rgb(226,174,53)" rx="2" ry="2" />
<text  x="37.17" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (58 samples, 0.02%)</title><rect x="210.2" y="437" width="0.2" height="15.0" fill="rgb(248,13,24)" rx="2" ry="2" />
<text  x="213.16" y="447.5" ></text>
</g>
<g >
<title>runtime.adjustpointers (59 samples, 0.02%)</title><rect x="1148.3" y="709" width="0.3" height="15.0" fill="rgb(253,130,5)" rx="2" ry="2" />
<text  x="1151.34" y="719.5" ></text>
</g>
<g >
<title>runtime.interhash (52 samples, 0.02%)</title><rect x="218.5" y="741" width="0.2" height="15.0" fill="rgb(253,201,24)" rx="2" ry="2" />
<text  x="221.47" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*protocol).Parse (40 samples, 0.01%)</title><rect x="720.1" y="757" width="0.2" height="15.0" fill="rgb(208,40,0)" rx="2" ry="2" />
<text  x="723.11" y="767.5" ></text>
</g>
<g >
<title>runtime.callers.func1 (46 samples, 0.01%)</title><rect x="767.9" y="533" width="0.2" height="15.0" fill="rgb(236,12,31)" rx="2" ry="2" />
<text  x="770.93" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleTimeWaitSegments.func1 (94 samples, 0.03%)</title><rect x="1069.4" y="853" width="0.4" height="15.0" fill="rgb(250,194,51)" rx="2" ry="2" />
<text  x="1072.41" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (55 samples, 0.02%)</title><rect x="16.9" y="837" width="0.2" height="15.0" fill="rgb(218,124,12)" rx="2" ry="2" />
<text  x="19.92" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut.func1 (76 samples, 0.02%)</title><rect x="176.4" y="613" width="0.3" height="15.0" fill="rgb(205,128,54)" rx="2" ry="2" />
<text  x="179.38" y="623.5" ></text>
</g>
<g >
<title>runtime.walltime1 (531 samples, 0.16%)</title><rect x="29.5" y="869" width="1.9" height="15.0" fill="rgb(206,112,45)" rx="2" ry="2" />
<text  x="32.51" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,454 samples, 0.45%)</title><rect x="913.5" y="677" width="5.2" height="15.0" fill="rgb(210,156,4)" rx="2" ry="2" />
<text  x="916.47" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut (183 samples, 0.06%)</title><rect x="176.3" y="645" width="0.6" height="15.0" fill="rgb(225,97,5)" rx="2" ry="2" />
<text  x="179.27" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).handleRcvdSegment (3,166 samples, 0.97%)</title><rect x="1077.5" y="805" width="11.5" height="15.0" fill="rgb(219,21,27)" rx="2" ry="2" />
<text  x="1080.54" y="815.5" ></text>
</g>
<g >
<title>runtime.procyield (43 samples, 0.01%)</title><rect x="844.6" y="741" width="0.2" height="15.0" fill="rgb(238,112,27)" rx="2" ry="2" />
<text  x="847.64" y="751.5" ></text>
</g>
<g >
<title>runtime.gcMarkDone (59 samples, 0.02%)</title><rect x="1111.0" y="853" width="0.2" height="15.0" fill="rgb(218,12,7)" rx="2" ry="2" />
<text  x="1114.04" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NUDState).ReachableTime (34 samples, 0.01%)</title><rect x="1084.6" y="725" width="0.1" height="15.0" fill="rgb(238,47,28)" rx="2" ry="2" />
<text  x="1087.56" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (10,819 samples, 3.32%)</title><rect x="957.6" y="517" width="39.1" height="15.0" fill="rgb(244,108,6)" rx="2" ry="2" />
<text  x="960.57" y="527.5" >[[k..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (232 samples, 0.07%)</title><rect x="909.9" y="549" width="0.9" height="15.0" fill="rgb(230,29,1)" rx="2" ry="2" />
<text  x="912.94" y="559.5" ></text>
</g>
<g >
<title>runtime/internal/atomic.Xchg64 (42 samples, 0.01%)</title><rect x="922.3" y="805" width="0.2" height="15.0" fill="rgb(245,128,35)" rx="2" ry="2" />
<text  x="925.35" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (45 samples, 0.01%)</title><rect x="18.0" y="709" width="0.2" height="15.0" fill="rgb(241,66,7)" rx="2" ry="2" />
<text  x="21.02" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,809 samples, 0.55%)</title><rect x="440.1" y="613" width="6.6" height="15.0" fill="rgb(237,183,50)" rx="2" ry="2" />
<text  x="443.11" y="623.5" ></text>
</g>
<g >
<title>runtime.notewakeup (2,797 samples, 0.86%)</title><rect x="748.0" y="517" width="10.1" height="15.0" fill="rgb(244,68,1)" rx="2" ry="2" />
<text  x="750.99" y="527.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).Deactivate (315 samples, 0.10%)</title><rect x="84.1" y="693" width="1.1" height="15.0" fill="rgb(222,164,12)" rx="2" ry="2" />
<text  x="87.10" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (66 samples, 0.02%)</title><rect x="13.2" y="581" width="0.2" height="15.0" fill="rgb(248,61,18)" rx="2" ry="2" />
<text  x="16.15" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).HandlePacket (92 samples, 0.03%)</title><rect x="19.7" y="837" width="0.3" height="15.0" fill="rgb(241,127,35)" rx="2" ry="2" />
<text  x="22.66" y="847.5" ></text>
</g>
<g >
<title>runtime.adjusttimers (897 samples, 0.27%)</title><rect x="94.0" y="629" width="3.3" height="15.0" fill="rgb(231,61,13)" rx="2" ry="2" />
<text  x="97.02" y="639.5" ></text>
</g>
<g >
<title>runtime.exitsyscall (36 samples, 0.01%)</title><rect x="845.6" y="821" width="0.1" height="15.0" fill="rgb(244,189,28)" rx="2" ry="2" />
<text  x="848.61" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (33 samples, 0.01%)</title><rect x="962.8" y="389" width="0.1" height="15.0" fill="rgb(251,227,17)" rx="2" ry="2" />
<text  x="965.77" y="399.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (122 samples, 0.04%)</title><rect x="1028.3" y="437" width="0.5" height="15.0" fill="rgb(230,78,33)" rx="2" ry="2" />
<text  x="1031.33" y="447.5" ></text>
</g>
<g >
<title>runtime.wakep (208 samples, 0.06%)</title><rect x="702.1" y="773" width="0.8" height="15.0" fill="rgb(238,225,34)" rx="2" ry="2" />
<text  x="705.12" y="783.5" ></text>
</g>
<g >
<title>runtime.runqgrab (42 samples, 0.01%)</title><rect x="905.6" y="741" width="0.2" height="15.0" fill="rgb(236,18,12)" rx="2" ry="2" />
<text  x="908.61" y="751.5" ></text>
</g>
<g >
<title>time.NewTimer (364 samples, 0.11%)</title><rect x="1048.4" y="709" width="1.4" height="15.0" fill="rgb(225,195,7)" rx="2" ry="2" />
<text  x="1051.44" y="719.5" ></text>
</g>
<g >
<title>runtime.convTslice (324 samples, 0.10%)</title><rect x="763.0" y="597" width="1.1" height="15.0" fill="rgb(238,72,20)" rx="2" ry="2" />
<text  x="765.96" y="607.5" ></text>
</g>
<g >
<title>runtime.newobject (47 samples, 0.01%)</title><rect x="216.4" y="709" width="0.2" height="15.0" fill="rgb(227,95,1)" rx="2" ry="2" />
<text  x="219.41" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (54 samples, 0.02%)</title><rect x="701.8" y="661" width="0.2" height="15.0" fill="rgb(234,109,28)" rx="2" ry="2" />
<text  x="704.80" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (386 samples, 0.12%)</title><rect x="1121.8" y="741" width="1.3" height="15.0" fill="rgb(225,129,33)" rx="2" ry="2" />
<text  x="1124.75" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (13,591 samples, 4.17%)</title><rect x="947.6" y="613" width="49.1" height="15.0" fill="rgb(249,132,35)" rx="2" ry="2" />
<text  x="950.56" y="623.5" >[[ke..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.x86FPState.FloatingPointData (60 samples, 0.02%)</title><rect x="227.6" y="805" width="0.2" height="15.0" fill="rgb(251,162,14)" rx="2" ry="2" />
<text  x="230.56" y="815.5" ></text>
</g>
<g >
<title>runtime.notesleep (75 samples, 0.02%)</title><rect x="17.9" y="773" width="0.3" height="15.0" fill="rgb(241,108,41)" rx="2" ry="2" />
<text  x="20.91" y="783.5" ></text>
</g>
<g >
<title>sync.(*poolChain).popTail (74 samples, 0.02%)</title><rect x="71.0" y="677" width="0.3" height="15.0" fill="rgb(233,171,25)" rx="2" ry="2" />
<text  x="74.02" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (321 samples, 0.10%)</title><rect x="1063.6" y="613" width="1.2" height="15.0" fill="rgb(218,145,10)" rx="2" ry="2" />
<text  x="1066.64" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*globalDirentMap).remove (74 samples, 0.02%)</title><rect x="64.4" y="645" width="0.2" height="15.0" fill="rgb(228,97,39)" rx="2" ry="2" />
<text  x="67.36" y="655.5" ></text>
</g>
<g >
<title>time.AfterFunc (128 samples, 0.04%)</title><rect x="1040.7" y="757" width="0.4" height="15.0" fill="rgb(207,62,6)" rx="2" ry="2" />
<text  x="1043.67" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="847.9" y="709" width="0.2" height="15.0" fill="rgb(250,228,1)" rx="2" ry="2" />
<text  x="850.90" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/fsutil.(*CachingInodeOperations).Write (2,994 samples, 0.92%)</title><rect x="191.9" y="693" width="10.9" height="15.0" fill="rgb(246,172,23)" rx="2" ry="2" />
<text  x="194.93" y="703.5" ></text>
</g>
<g >
<title>time.startTimer (32 samples, 0.01%)</title><rect x="1019.7" y="805" width="0.1" height="15.0" fill="rgb(210,82,12)" rx="2" ry="2" />
<text  x="1022.72" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (184 samples, 0.06%)</title><rect x="1119.1" y="757" width="0.6" height="15.0" fill="rgb(209,63,48)" rx="2" ry="2" />
<text  x="1122.06" y="767.5" ></text>
</g>
<g >
<title>runtime.exitsyscall (29 samples, 0.01%)</title><rect x="201.7" y="485" width="0.1" height="15.0" fill="rgb(240,219,25)" rx="2" ry="2" />
<text  x="204.72" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).MaxHeaderLength (37 samples, 0.01%)</title><rect x="180.7" y="549" width="0.1" height="15.0" fill="rgb(223,31,12)" rx="2" ry="2" />
<text  x="183.67" y="559.5" ></text>
</g>
<g >
<title>runtime.checkTimers (1,067 samples, 0.33%)</title><rect x="93.6" y="645" width="3.8" height="15.0" fill="rgb(205,59,4)" rx="2" ry="2" />
<text  x="96.56" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,931 samples, 0.59%)</title><rect x="911.7" y="693" width="7.0" height="15.0" fill="rgb(214,63,5)" rx="2" ry="2" />
<text  x="914.74" y="703.5" ></text>
</g>
<g >
<title>runtime.newproc.func1 (242 samples, 0.07%)</title><rect x="1080.1" y="629" width="0.9" height="15.0" fill="rgb(248,216,43)" rx="2" ry="2" />
<text  x="1083.14" y="639.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (48 samples, 0.01%)</title><rect x="927.1" y="757" width="0.2" height="15.0" fill="rgb(229,88,22)" rx="2" ry="2" />
<text  x="930.11" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).IsBroadcast (28 samples, 0.01%)</title><rect x="1033.3" y="725" width="0.1" height="15.0" fill="rgb(236,58,30)" rx="2" ry="2" />
<text  x="1036.33" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/rawfile.NonBlockingSendMMsg (36 samples, 0.01%)</title><rect x="926.3" y="805" width="0.1" height="15.0" fill="rgb(215,158,10)" rx="2" ry="2" />
<text  x="929.31" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*endpoint).WritePackets (20,962 samples, 6.42%)</title><rect x="922.8" y="837" width="75.8" height="15.0" fill="rgb(238,159,5)" rx="2" ry="2" />
<text  x="925.80" y="847.5" >gvisor.d..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*protocol).ParseAddresses (138 samples, 0.04%)</title><rect x="740.0" y="693" width="0.5" height="15.0" fill="rgb(250,198,36)" rx="2" ry="2" />
<text  x="743.01" y="703.5" ></text>
</g>
<g >
<title>runtime.bgsweep (444 samples, 0.14%)</title><rect x="1109.4" y="869" width="1.6" height="15.0" fill="rgb(231,83,26)" rx="2" ry="2" />
<text  x="1112.41" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (188 samples, 0.06%)</title><rect x="115.2" y="517" width="0.7" height="15.0" fill="rgb(212,199,51)" rx="2" ry="2" />
<text  x="118.20" y="527.5" ></text>
</g>
<g >
<title>runtime.unlock2 (76 samples, 0.02%)</title><rect x="148.7" y="629" width="0.2" height="15.0" fill="rgb(237,75,52)" rx="2" ry="2" />
<text  x="151.66" y="639.5" ></text>
</g>
<g >
<title>runtime.findrunnable (240 samples, 0.07%)</title><rect x="1006.8" y="757" width="0.9" height="15.0" fill="rgb(209,102,33)" rx="2" ry="2" />
<text  x="1009.79" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (156 samples, 0.05%)</title><rect x="22.7" y="757" width="0.6" height="15.0" fill="rgb(210,70,9)" rx="2" ry="2" />
<text  x="25.71" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).MaxHeaderLength (39 samples, 0.01%)</title><rect x="1090.5" y="741" width="0.2" height="15.0" fill="rgb(231,173,38)" rx="2" ry="2" />
<text  x="1093.51" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (51 samples, 0.02%)</title><rect x="13.2" y="565" width="0.2" height="15.0" fill="rgb(249,52,42)" rx="2" ry="2" />
<text  x="16.21" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,104 samples, 0.34%)</title><rect x="914.7" y="613" width="4.0" height="15.0" fill="rgb(221,145,13)" rx="2" ry="2" />
<text  x="917.74" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,490 samples, 2.30%)</title><rect x="967.5" y="421" width="27.1" height="15.0" fill="rgb(250,160,30)" rx="2" ry="2" />
<text  x="970.49" y="431.5" >[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (30 samples, 0.01%)</title><rect x="725.2" y="693" width="0.1" height="15.0" fill="rgb(216,122,49)" rx="2" ry="2" />
<text  x="728.22" y="703.5" ></text>
</g>
<g >
<title>runtime.chansend (569 samples, 0.17%)</title><rect x="1002.0" y="757" width="2.0" height="15.0" fill="rgb(207,173,31)" rx="2" ry="2" />
<text  x="1004.97" y="767.5" ></text>
</g>
<g >
<title>runtime.notesleep (68 samples, 0.02%)</title><rect x="1189.7" y="789" width="0.3" height="15.0" fill="rgb(225,167,13)" rx="2" ry="2" />
<text  x="1192.75" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.PseudoHeaderChecksum (39 samples, 0.01%)</title><rect x="214.9" y="565" width="0.2" height="15.0" fill="rgb(219,109,51)" rx="2" ry="2" />
<text  x="217.95" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (163 samples, 0.05%)</title><rect x="115.3" y="501" width="0.6" height="15.0" fill="rgb(224,158,24)" rx="2" ry="2" />
<text  x="118.29" y="511.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (66 samples, 0.02%)</title><rect x="1069.4" y="837" width="0.3" height="15.0" fill="rgb(209,222,20)" rx="2" ry="2" />
<text  x="1072.42" y="847.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (113 samples, 0.03%)</title><rect x="762.1" y="597" width="0.4" height="15.0" fill="rgb(218,10,39)" rx="2" ry="2" />
<text  x="765.08" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).WritePacket (197 samples, 0.06%)</title><rect x="1091.3" y="645" width="0.7" height="15.0" fill="rgb(244,157,20)" rx="2" ry="2" />
<text  x="1094.27" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.NewSubnet (42 samples, 0.01%)</title><rect x="731.1" y="709" width="0.1" height="15.0" fill="rgb(254,176,0)" rx="2" ry="2" />
<text  x="734.09" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.buildTCPHdr (146 samples, 0.04%)</title><rect x="1030.5" y="741" width="0.5" height="15.0" fill="rgb(252,149,28)" rx="2" ry="2" />
<text  x="1033.47" y="751.5" ></text>
</g>
<g >
<title>runtime.notewakeup (107 samples, 0.03%)</title><rect x="19.0" y="821" width="0.4" height="15.0" fill="rgb(236,154,19)" rx="2" ry="2" />
<text  x="22.03" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).addIPHeader (74 samples, 0.02%)</title><rect x="1027.2" y="709" width="0.3" height="15.0" fill="rgb(222,9,30)" rx="2" ry="2" />
<text  x="1030.23" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (399 samples, 0.12%)</title><rect x="705.3" y="629" width="1.4" height="15.0" fill="rgb(248,110,30)" rx="2" ry="2" />
<text  x="708.25" y="639.5" ></text>
</g>
<g >
<title>runtime.usleep (946 samples, 0.29%)</title><rect x="907.4" y="709" width="3.4" height="15.0" fill="rgb(241,110,46)" rx="2" ry="2" />
<text  x="910.36" y="719.5" ></text>
</g>
<g >
<title>runtime.makeslice (431 samples, 0.13%)</title><rect x="710.2" y="805" width="1.6" height="15.0" fill="rgb(243,170,7)" rx="2" ry="2" />
<text  x="713.23" y="815.5" ></text>
</g>
<g >
<title>runtime.epollwait (253 samples, 0.08%)</title><rect x="1118.8" y="805" width="0.9" height="15.0" fill="rgb(245,0,32)" rx="2" ry="2" />
<text  x="1121.81" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).enqueuePacketBuffer (476 samples, 0.15%)</title><rect x="1073.6" y="645" width="1.7" height="15.0" fill="rgb(240,155,23)" rx="2" ry="2" />
<text  x="1076.57" y="655.5" ></text>
</g>
<g >
<title>runtime.(*waitq).dequeue (206 samples, 0.06%)</title><rect x="1002.3" y="741" width="0.8" height="15.0" fill="rgb(213,187,27)" rx="2" ry="2" />
<text  x="1005.31" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (158 samples, 0.05%)</title><rect x="956.0" y="389" width="0.5" height="15.0" fill="rgb(218,201,22)" rx="2" ry="2" />
<text  x="958.97" y="399.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (45 samples, 0.01%)</title><rect x="724.8" y="693" width="0.2" height="15.0" fill="rgb(244,13,47)" rx="2" ry="2" />
<text  x="727.83" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Close (1,709 samples, 0.52%)</title><rect x="63.4" y="773" width="6.2" height="15.0" fill="rgb(230,145,37)" rx="2" ry="2" />
<text  x="66.40" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (241 samples, 0.07%)</title><rect x="1082.6" y="565" width="0.9" height="15.0" fill="rgb(220,85,39)" rx="2" ry="2" />
<text  x="1085.59" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.newIncomingSegment (2,016 samples, 0.62%)</title><rect x="761.4" y="629" width="7.3" height="15.0" fill="rgb(227,175,47)" rx="2" ry="2" />
<text  x="764.41" y="639.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (28 samples, 0.01%)</title><rect x="206.9" y="613" width="0.1" height="15.0" fill="rgb(247,15,35)" rx="2" ry="2" />
<text  x="209.90" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (28 samples, 0.01%)</title><rect x="779.6" y="693" width="0.1" height="15.0" fill="rgb(243,123,10)" rx="2" ry="2" />
<text  x="782.57" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*endpointsByNIC).registerEndpoint (187 samples, 0.06%)</title><rect x="1022.7" y="773" width="0.7" height="15.0" fill="rgb(222,30,13)" rx="2" ry="2" />
<text  x="1025.69" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).resetKickerLocked (33 samples, 0.01%)</title><rect x="1079.4" y="645" width="0.1" height="15.0" fill="rgb(244,199,7)" rx="2" ry="2" />
<text  x="1082.37" y="655.5" ></text>
</g>
<g >
<title>runtime.mallocgc (28 samples, 0.01%)</title><rect x="717.3" y="805" width="0.1" height="15.0" fill="rgb(219,170,36)" rx="2" ry="2" />
<text  x="720.27" y="815.5" ></text>
</g>
<g >
<title>runtime.isSystemGoroutine (28 samples, 0.01%)</title><rect x="1048.3" y="661" width="0.1" height="15.0" fill="rgb(217,73,21)" rx="2" ry="2" />
<text  x="1051.28" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*handleReadWriter).WriteFromBlocks (47 samples, 0.01%)</title><rect x="21.6" y="645" width="0.1" height="15.0" fill="rgb(210,75,36)" rx="2" ry="2" />
<text  x="24.57" y="655.5" ></text>
</g>
<g >
<title>runtime.mallocgc (290 samples, 0.09%)</title><rect x="172.1" y="725" width="1.0" height="15.0" fill="rgb(243,144,48)" rx="2" ry="2" />
<text  x="175.05" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (318 samples, 0.10%)</title><rect x="1122.0" y="693" width="1.1" height="15.0" fill="rgb(223,99,21)" rx="2" ry="2" />
<text  x="1125.00" y="703.5" ></text>
</g>
<g >
<title>runtime.resetspinning (396 samples, 0.12%)</title><rect x="1065.1" y="757" width="1.4" height="15.0" fill="rgb(217,32,16)" rx="2" ry="2" />
<text  x="1068.11" y="767.5" ></text>
</g>
<g >
<title>runtime.isSystemGoroutine (89 samples, 0.03%)</title><rect x="1080.5" y="597" width="0.4" height="15.0" fill="rgb(208,15,15)" rx="2" ry="2" />
<text  x="1083.55" y="607.5" ></text>
</g>
<g >
<title>runtime.makechan (138 samples, 0.04%)</title><rect x="1081.1" y="629" width="0.5" height="15.0" fill="rgb(243,98,33)" rx="2" ry="2" />
<text  x="1084.05" y="639.5" ></text>
</g>
<g >
<title>runtime.acquirep (32 samples, 0.01%)</title><rect x="107.3" y="629" width="0.1" height="15.0" fill="rgb(252,225,37)" rx="2" ry="2" />
<text  x="110.32" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (156 samples, 0.05%)</title><rect x="205.9" y="597" width="0.6" height="15.0" fill="rgb(206,117,47)" rx="2" ry="2" />
<text  x="208.91" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (177 samples, 0.05%)</title><rect x="1060.2" y="709" width="0.6" height="15.0" fill="rgb(207,35,31)" rx="2" ry="2" />
<text  x="1063.16" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).StateData (30 samples, 0.01%)</title><rect x="223.5" y="821" width="0.1" height="15.0" fill="rgb(253,216,2)" rx="2" ry="2" />
<text  x="226.46" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,500 samples, 0.46%)</title><rect x="195.8" y="373" width="5.4" height="15.0" fill="rgb(213,206,0)" rx="2" ry="2" />
<text  x="198.76" y="383.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).Notify (172 samples, 0.05%)</title><rect x="1071.3" y="757" width="0.6" height="15.0" fill="rgb(250,180,53)" rx="2" ry="2" />
<text  x="1074.30" y="767.5" ></text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (43 samples, 0.01%)</title><rect x="173.1" y="725" width="0.2" height="15.0" fill="rgb(227,215,34)" rx="2" ry="2" />
<text  x="176.10" y="735.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (33 samples, 0.01%)</title><rect x="1086.7" y="693" width="0.2" height="15.0" fill="rgb(224,57,34)" rx="2" ry="2" />
<text  x="1089.75" y="703.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast32 (28 samples, 0.01%)</title><rect x="1009.9" y="741" width="0.1" height="15.0" fill="rgb(237,74,36)" rx="2" ry="2" />
<text  x="1012.86" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*listenContext).removePendingEndpoint (147 samples, 0.05%)</title><rect x="1020.5" y="837" width="0.5" height="15.0" fill="rgb(214,126,21)" rx="2" ry="2" />
<text  x="1023.50" y="847.5" ></text>
</g>
<g >
<title>runtime.runqgrab (2,238 samples, 0.69%)</title><rect x="119.3" y="613" width="8.1" height="15.0" fill="rgb(239,31,36)" rx="2" ry="2" />
<text  x="122.28" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="196.0" y="309" width="0.2" height="15.0" fill="rgb(205,203,37)" rx="2" ry="2" />
<text  x="199.05" y="319.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).Fetch (2,903 samples, 0.89%)</title><rect x="1056.6" y="837" width="10.5" height="15.0" fill="rgb(223,57,25)" rx="2" ry="2" />
<text  x="1059.57" y="847.5" ></text>
</g>
<g >
<title>runtime.(*mheap).alloc.func1 (44 samples, 0.01%)</title><rect x="1042.4" y="661" width="0.2" height="15.0" fill="rgb(228,225,33)" rx="2" ry="2" />
<text  x="1045.43" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*protocol).Parse (190 samples, 0.06%)</title><rect x="780.4" y="741" width="0.6" height="15.0" fill="rgb(216,5,21)" rx="2" ry="2" />
<text  x="783.35" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (188 samples, 0.06%)</title><rect x="700.9" y="661" width="0.7" height="15.0" fill="rgb(229,68,32)" rx="2" ry="2" />
<text  x="703.91" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).findLocalRouteRLocked (957 samples, 0.29%)</title><rect x="1034.5" y="773" width="3.5" height="15.0" fill="rgb(220,198,28)" rx="2" ry="2" />
<text  x="1037.53" y="783.5" ></text>
</g>
<g >
<title>nf_conntrack_in (233 samples, 0.07%)</title><rect x="991.2" y="181" width="0.9" height="15.0" fill="rgb(252,78,49)" rx="2" ry="2" />
<text  x="994.22" y="191.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (655 samples, 0.20%)</title><rect x="1124.4" y="741" width="2.4" height="15.0" fill="rgb(222,182,47)" rx="2" ry="2" />
<text  x="1127.42" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*packetEndpointList).forEach (177 samples, 0.05%)</title><rect x="792.9" y="741" width="0.6" height="15.0" fill="rgb(253,51,36)" rx="2" ry="2" />
<text  x="795.87" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (80 samples, 0.02%)</title><rect x="701.3" y="581" width="0.3" height="15.0" fill="rgb(253,172,35)" rx="2" ry="2" />
<text  x="704.30" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).DeliverOutboundPacket (49 samples, 0.02%)</title><rect x="1091.4" y="597" width="0.2" height="15.0" fill="rgb(246,73,54)" rx="2" ry="2" />
<text  x="1094.38" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (116 samples, 0.04%)</title><rect x="13.0" y="693" width="0.4" height="15.0" fill="rgb(220,5,5)" rx="2" ry="2" />
<text  x="15.97" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/fsutil.(*CachingInodeOperations).Write (47 samples, 0.01%)</title><rect x="21.6" y="757" width="0.1" height="15.0" fill="rgb(210,121,19)" rx="2" ry="2" />
<text  x="24.57" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (173 samples, 0.05%)</title><rect x="1065.9" y="565" width="0.6" height="15.0" fill="rgb(221,74,0)" rx="2" ry="2" />
<text  x="1068.90" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,581 samples, 0.48%)</title><rect x="889.2" y="645" width="5.7" height="15.0" fill="rgb(238,146,34)" rx="2" ry="2" />
<text  x="892.17" y="655.5" ></text>
</g>
<g >
<title>runtime.wakep (52 samples, 0.02%)</title><rect x="1109.1" y="773" width="0.2" height="15.0" fill="rgb(205,185,49)" rx="2" ry="2" />
<text  x="1112.11" y="783.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (169 samples, 0.05%)</title><rect x="711.1" y="773" width="0.6" height="15.0" fill="rgb(218,212,30)" rx="2" ry="2" />
<text  x="714.12" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (248 samples, 0.08%)</title><rect x="1118.8" y="789" width="0.9" height="15.0" fill="rgb(210,72,31)" rx="2" ry="2" />
<text  x="1121.83" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (201 samples, 0.06%)</title><rect x="700.9" y="677" width="0.7" height="15.0" fill="rgb(215,108,6)" rx="2" ry="2" />
<text  x="703.86" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (239 samples, 0.07%)</title><rect x="1065.7" y="613" width="0.8" height="15.0" fill="rgb(244,161,21)" rx="2" ry="2" />
<text  x="1068.66" y="623.5" ></text>
</g>
<g >
<title>runtime.lock2 (57 samples, 0.02%)</title><rect x="824.0" y="741" width="0.2" height="15.0" fill="rgb(228,87,1)" rx="2" ry="2" />
<text  x="826.95" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/secio.(*SectionWriter).Write (2,336 samples, 0.72%)</title><rect x="193.4" y="533" width="8.4" height="15.0" fill="rgb(253,90,5)" rx="2" ry="2" />
<text  x="196.38" y="543.5" ></text>
</g>
<g >
<title>runtime.(*lfstack).push (143 samples, 0.04%)</title><rect x="1146.5" y="757" width="0.5" height="15.0" fill="rgb(230,17,6)" rx="2" ry="2" />
<text  x="1149.53" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,801 samples, 0.55%)</title><rect x="815.6" y="661" width="6.6" height="15.0" fill="rgb(238,177,12)" rx="2" ry="2" />
<text  x="818.65" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (15,187 samples, 4.65%)</title><rect x="640.7" y="805" width="54.9" height="15.0" fill="rgb(243,145,20)" rx="2" ry="2" />
<text  x="643.69" y="815.5" >[[ker..</text>
</g>
<g >
<title>runtime.futex (2,754 samples, 0.84%)</title><rect x="748.1" y="501" width="10.0" height="15.0" fill="rgb(234,56,1)" rx="2" ry="2" />
<text  x="751.11" y="511.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).Activate (279 samples, 0.09%)</title><rect x="80.3" y="677" width="1.0" height="15.0" fill="rgb(229,60,45)" rx="2" ry="2" />
<text  x="83.28" y="687.5" ></text>
</g>
<g >
<title>sync.(*RWMutex).RUnlock (35 samples, 0.01%)</title><rect x="780.2" y="725" width="0.1" height="15.0" fill="rgb(232,49,43)" rx="2" ry="2" />
<text  x="783.21" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (66 samples, 0.02%)</title><rect x="1189.8" y="693" width="0.2" height="15.0" fill="rgb(253,128,18)" rx="2" ry="2" />
<text  x="1192.75" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (169 samples, 0.05%)</title><rect x="701.0" y="629" width="0.6" height="15.0" fill="rgb(249,210,47)" rx="2" ry="2" />
<text  x="703.98" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (46 samples, 0.01%)</title><rect x="162.3" y="661" width="0.1" height="15.0" fill="rgb(217,93,25)" rx="2" ry="2" />
<text  x="165.27" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).doTimeWait (3,607 samples, 1.11%)</title><rect x="1056.3" y="853" width="13.0" height="15.0" fill="rgb(248,78,22)" rx="2" ry="2" />
<text  x="1059.29" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (419 samples, 0.13%)</title><rect x="1188.2" y="725" width="1.5" height="15.0" fill="rgb(233,153,41)" rx="2" ry="2" />
<text  x="1191.19" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,311 samples, 0.40%)</title><rect x="122.6" y="581" width="4.8" height="15.0" fill="rgb(210,194,30)" rx="2" ry="2" />
<text  x="125.63" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doSyscallEnter (50,200 samples, 15.38%)</title><rect x="39.3" y="821" width="181.5" height="15.0" fill="rgb(246,175,16)" rx="2" ry="2" />
<text  x="42.30" y="831.5" >gvisor.dev/gvisor/pkg/s..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (82 samples, 0.03%)</title><rect x="821.9" y="613" width="0.3" height="15.0" fill="rgb(213,16,51)" rx="2" ry="2" />
<text  x="824.86" y="623.5" ></text>
</g>
<g >
<title>runtime.systemstack (104 samples, 0.03%)</title><rect x="1048.1" y="709" width="0.3" height="15.0" fill="rgb(227,56,54)" rx="2" ry="2" />
<text  x="1051.07" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*WeakRef).zap (70 samples, 0.02%)</title><rect x="63.5" y="725" width="0.3" height="15.0" fill="rgb(207,225,17)" rx="2" ry="2" />
<text  x="66.53" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*pollEntry).WeakRefGone (361 samples, 0.11%)</title><rect x="67.2" y="725" width="1.3" height="15.0" fill="rgb(208,203,44)" rx="2" ry="2" />
<text  x="70.20" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (87 samples, 0.03%)</title><rect x="1108.7" y="629" width="0.3" height="15.0" fill="rgb(219,106,10)" rx="2" ry="2" />
<text  x="1111.72" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (19,981 samples, 6.12%)</title><rect x="565.8" y="709" width="72.2" height="15.0" fill="rgb(224,85,34)" rx="2" ry="2" />
<text  x="568.76" y="719.5" >[[kernel..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (741 samples, 0.23%)</title><rect x="908.1" y="677" width="2.7" height="15.0" fill="rgb(252,210,42)" rx="2" ry="2" />
<text  x="911.10" y="687.5" ></text>
</g>
<g >
<title>runtime.lock2 (89 samples, 0.03%)</title><rect x="1150.7" y="709" width="0.3" height="15.0" fill="rgb(243,119,38)" rx="2" ry="2" />
<text  x="1153.71" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (73 samples, 0.02%)</title><rect x="13.1" y="597" width="0.3" height="15.0" fill="rgb(211,131,49)" rx="2" ry="2" />
<text  x="16.13" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (156 samples, 0.05%)</title><rect x="451.7" y="597" width="0.5" height="15.0" fill="rgb(229,18,39)" rx="2" ry="2" />
<text  x="454.67" y="607.5" ></text>
</g>
<g >
<title>fdb_find_rcu (30 samples, 0.01%)</title><rect x="991.0" y="149" width="0.1" height="15.0" fill="rgb(207,62,0)" rx="2" ry="2" />
<text  x="993.96" y="159.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).AcquireAssignedAddressOrMatching (1,913 samples, 0.59%)</title><rect x="783.7" y="693" width="6.9" height="15.0" fill="rgb(246,4,52)" rx="2" ry="2" />
<text  x="786.67" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*lockedRandomSource).Int63 (35 samples, 0.01%)</title><rect x="1040.4" y="773" width="0.1" height="15.0" fill="rgb(211,83,7)" rx="2" ry="2" />
<text  x="1043.39" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.copyTimespecOut (592 samples, 0.18%)</title><rect x="61.1" y="757" width="2.1" height="15.0" fill="rgb(252,208,7)" rx="2" ry="2" />
<text  x="64.06" y="767.5" ></text>
</g>
<g >
<title>runtime.newobject (94 samples, 0.03%)</title><rect x="1033.9" y="709" width="0.3" height="15.0" fill="rgb(207,39,5)" rx="2" ry="2" />
<text  x="1036.88" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,276 samples, 0.39%)</title><rect x="151.7" y="549" width="4.6" height="15.0" fill="rgb(228,163,36)" rx="2" ry="2" />
<text  x="154.68" y="559.5" ></text>
</g>
<g >
<title>runtime.(*itabTableType).find (42 samples, 0.01%)</title><rect x="1035.6" y="709" width="0.2" height="15.0" fill="rgb(246,42,22)" rx="2" ry="2" />
<text  x="1038.65" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (300 samples, 0.09%)</title><rect x="909.7" y="565" width="1.1" height="15.0" fill="rgb(249,162,18)" rx="2" ry="2" />
<text  x="912.69" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (246 samples, 0.08%)</title><rect x="704.0" y="661" width="0.9" height="15.0" fill="rgb(213,229,54)" rx="2" ry="2" />
<text  x="706.99" y="671.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_faststr (72 samples, 0.02%)</title><rect x="737.6" y="677" width="0.2" height="15.0" fill="rgb(215,217,3)" rx="2" ry="2" />
<text  x="740.55" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (446 samples, 0.14%)</title><rect x="954.9" y="421" width="1.6" height="15.0" fill="rgb(251,92,25)" rx="2" ry="2" />
<text  x="957.93" y="431.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut (530 samples, 0.16%)</title><rect x="61.2" y="725" width="1.9" height="15.0" fill="rgb(218,162,50)" rx="2" ry="2" />
<text  x="64.22" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.New.func1 (28,711 samples, 8.80%)</title><rect x="895.7" y="869" width="103.9" height="15.0" fill="rgb(227,26,11)" rx="2" ry="2" />
<text  x="898.74" y="879.5" >gvisor.dev/g..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (120 samples, 0.04%)</title><rect x="157.4" y="501" width="0.4" height="15.0" fill="rgb(216,128,30)" rx="2" ry="2" />
<text  x="160.39" y="511.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*WeakRef).init (54 samples, 0.02%)</title><rect x="70.5" y="709" width="0.2" height="15.0" fill="rgb(222,96,16)" rx="2" ry="2" />
<text  x="73.49" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).get (87 samples, 0.03%)</title><rect x="1074.7" y="613" width="0.3" height="15.0" fill="rgb(245,188,24)" rx="2" ry="2" />
<text  x="1077.69" y="623.5" ></text>
</g>
<g >
<title>runtime.mallocgc (66 samples, 0.02%)</title><rect x="1083.9" y="661" width="0.2" height="15.0" fill="rgb(252,201,3)" rx="2" ry="2" />
<text  x="1086.90" y="671.5" ></text>
</g>
<g >
<title>runtime.systemstack (161 samples, 0.05%)</title><rect x="766.5" y="517" width="0.6" height="15.0" fill="rgb(239,66,4)" rx="2" ry="2" />
<text  x="769.49" y="527.5" ></text>
</g>
<g >
<title>runtime.checkTimers (1,953 samples, 0.60%)</title><rect x="107.5" y="629" width="7.0" height="15.0" fill="rgb(236,0,36)" rx="2" ry="2" />
<text  x="110.46" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).checkLimit (103 samples, 0.03%)</title><rect x="190.7" y="725" width="0.4" height="15.0" fill="rgb(248,12,20)" rx="2" ry="2" />
<text  x="193.74" y="735.5" ></text>
</g>
<g >
<title>runtime.notesleep (5,469 samples, 1.68%)</title><rect x="128.7" y="613" width="19.8" height="15.0" fill="rgb(226,124,36)" rx="2" ry="2" />
<text  x="131.71" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (28 samples, 0.01%)</title><rect x="726.0" y="645" width="0.1" height="15.0" fill="rgb(217,76,18)" rx="2" ry="2" />
<text  x="729.04" y="655.5" ></text>
</g>
<g >
<title>runtime.newobject (369 samples, 0.11%)</title><rect x="171.9" y="741" width="1.4" height="15.0" fill="rgb(207,121,38)" rx="2" ry="2" />
<text  x="174.92" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (146 samples, 0.04%)</title><rect x="198.2" y="309" width="0.5" height="15.0" fill="rgb(207,28,30)" rx="2" ry="2" />
<text  x="201.18" y="319.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (39 samples, 0.01%)</title><rect x="895.4" y="837" width="0.2" height="15.0" fill="rgb(212,66,44)" rx="2" ry="2" />
<text  x="898.42" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.accept (4,236 samples, 1.30%)</title><rect x="43.5" y="757" width="15.3" height="15.0" fill="rgb(237,189,47)" rx="2" ry="2" />
<text  x="46.47" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (118 samples, 0.04%)</title><rect x="157.4" y="453" width="0.4" height="15.0" fill="rgb(224,105,33)" rx="2" ry="2" />
<text  x="160.39" y="463.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (164 samples, 0.05%)</title><rect x="711.1" y="757" width="0.6" height="15.0" fill="rgb(243,64,38)" rx="2" ry="2" />
<text  x="714.12" y="767.5" ></text>
</g>
<g >
<title>runtime.mallocgc (111 samples, 0.03%)</title><rect x="730.0" y="677" width="0.4" height="15.0" fill="rgb(237,13,4)" rx="2" ry="2" />
<text  x="732.98" y="687.5" ></text>
</g>
<g >
<title>runtime.unlock2 (109 samples, 0.03%)</title><rect x="160.9" y="677" width="0.4" height="15.0" fill="rgb(242,108,43)" rx="2" ry="2" />
<text  x="163.90" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacket (955 samples, 0.29%)</title><rect x="210.0" y="485" width="3.4" height="15.0" fill="rgb(232,158,23)" rx="2" ry="2" />
<text  x="212.95" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyInIovecs (218 samples, 0.07%)</title><rect x="203.8" y="741" width="0.7" height="15.0" fill="rgb(218,130,7)" rx="2" ry="2" />
<text  x="206.76" y="751.5" ></text>
</g>
<g >
<title>veth_xmit (393 samples, 0.12%)</title><rect x="996.7" y="645" width="1.4" height="15.0" fill="rgb(222,100,3)" rx="2" ry="2" />
<text  x="999.71" y="655.5" ></text>
</g>
<g >
<title>runtime.getStackMap (222 samples, 0.07%)</title><rect x="1103.0" y="693" width="0.8" height="15.0" fill="rgb(239,77,45)" rx="2" ry="2" />
<text  x="1105.98" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/interrupt.(*Forwarder).Disable (183 samples, 0.06%)</title><rect x="227.8" y="805" width="0.6" height="15.0" fill="rgb(218,187,53)" rx="2" ry="2" />
<text  x="230.77" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/marshal/primitive.CopyUint32In (139 samples, 0.04%)</title><rect x="56.6" y="725" width="0.6" height="15.0" fill="rgb(225,38,38)" rx="2" ry="2" />
<text  x="59.65" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).getSendBufferSize (30 samples, 0.01%)</title><rect x="207.2" y="661" width="0.1" height="15.0" fill="rgb(209,72,29)" rx="2" ry="2" />
<text  x="210.16" y="671.5" ></text>
</g>
<g >
<title>runtime.runqget (74 samples, 0.02%)</title><rect x="156.8" y="645" width="0.3" height="15.0" fill="rgb(246,127,42)" rx="2" ry="2" />
<text  x="159.81" y="655.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (37 samples, 0.01%)</title><rect x="207.4" y="581" width="0.2" height="15.0" fill="rgb(214,65,42)" rx="2" ry="2" />
<text  x="210.42" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).DecRef (236 samples, 0.07%)</title><rect x="776.2" y="709" width="0.8" height="15.0" fill="rgb(233,168,21)" rx="2" ry="2" />
<text  x="779.17" y="719.5" ></text>
</g>
<g >
<title>runtime.gosched_m (248 samples, 0.08%)</title><rect x="1129.0" y="853" width="0.9" height="15.0" fill="rgb(227,184,29)" rx="2" ry="2" />
<text  x="1131.97" y="863.5" ></text>
</g>
<g >
<title>encoding/binary.(*littleEndian).PutUint16 (39 samples, 0.01%)</title><rect x="924.5" y="773" width="0.1" height="15.0" fill="rgb(252,20,34)" rx="2" ry="2" />
<text  x="927.49" y="783.5" ></text>
</g>
<g >
<title>runtime.newobject (79 samples, 0.02%)</title><rect x="1029.7" y="645" width="0.3" height="15.0" fill="rgb(242,163,14)" rx="2" ry="2" />
<text  x="1032.68" y="655.5" ></text>
</g>
<g >
<title>runtime.addtimer (461 samples, 0.14%)</title><rect x="1081.8" y="613" width="1.7" height="15.0" fill="rgb(219,81,41)" rx="2" ry="2" />
<text  x="1084.80" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (124 samples, 0.04%)</title><rect x="157.4" y="565" width="0.4" height="15.0" fill="rgb(232,128,28)" rx="2" ry="2" />
<text  x="160.37" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).block (22,776 samples, 6.98%)</title><rect x="79.6" y="725" width="82.3" height="15.0" fill="rgb(216,150,3)" rx="2" ry="2" />
<text  x="82.55" y="735.5" >gvisor.de..</text>
</g>
<g >
<title>runtime.mapaccess2_fast32 (236 samples, 0.07%)</title><rect x="794.3" y="741" width="0.9" height="15.0" fill="rgb(218,83,32)" rx="2" ry="2" />
<text  x="797.31" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (266 samples, 0.08%)</title><rect x="212.2" y="181" width="0.9" height="15.0" fill="rgb(211,25,38)" rx="2" ry="2" />
<text  x="215.15" y="191.5" ></text>
</g>
<g >
<title>runtime.futex (70 samples, 0.02%)</title><rect x="1153.3" y="693" width="0.2" height="15.0" fill="rgb(233,119,35)" rx="2" ry="2" />
<text  x="1156.26" y="703.5" ></text>
</g>
<g >
<title>runtime.mapdelete (40 samples, 0.01%)</title><rect x="1099.9" y="805" width="0.2" height="15.0" fill="rgb(212,130,5)" rx="2" ry="2" />
<text  x="1102.94" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="1185.9" y="741" width="0.1" height="15.0" fill="rgb(243,148,1)" rx="2" ry="2" />
<text  x="1188.87" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="905.4" y="661" width="0.1" height="15.0" fill="rgb(243,74,15)" rx="2" ry="2" />
<text  x="908.41" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,239 samples, 0.38%)</title><rect x="914.2" y="645" width="4.5" height="15.0" fill="rgb(250,29,1)" rx="2" ry="2" />
<text  x="917.25" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Write (52 samples, 0.02%)</title><rect x="21.6" y="837" width="0.2" height="15.0" fill="rgb(234,46,43)" rx="2" ry="2" />
<text  x="24.57" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (46 samples, 0.01%)</title><rect x="1120.2" y="725" width="0.2" height="15.0" fill="rgb(234,120,39)" rx="2" ry="2" />
<text  x="1123.22" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (127 samples, 0.04%)</title><rect x="732.8" y="693" width="0.4" height="15.0" fill="rgb(232,97,41)" rx="2" ry="2" />
<text  x="735.79" y="703.5" ></text>
</g>
<g >
<title>runtime.stackalloc (101 samples, 0.03%)</title><rect x="1017.9" y="741" width="0.3" height="15.0" fill="rgb(230,111,6)" rx="2" ry="2" />
<text  x="1020.88" y="751.5" ></text>
</g>
<g >
<title>runtime.siftdownTimer (28 samples, 0.01%)</title><rect x="903.9" y="677" width="0.1" height="15.0" fill="rgb(247,189,8)" rx="2" ry="2" />
<text  x="906.88" y="687.5" ></text>
</g>
<g >
<title>[[vdso]] (435 samples, 0.13%)</title><rect x="29.9" y="853" width="1.5" height="15.0" fill="rgb(221,23,44)" rx="2" ry="2" />
<text  x="32.86" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (485 samples, 0.15%)</title><rect x="920.1" y="661" width="1.8" height="15.0" fill="rgb(223,173,38)" rx="2" ry="2" />
<text  x="923.12" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,446 samples, 0.75%)</title><rect x="139.6" y="469" width="8.8" height="15.0" fill="rgb(208,16,23)" rx="2" ry="2" />
<text  x="142.57" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut.func1 (319 samples, 0.10%)</title><rect x="74.4" y="693" width="1.1" height="15.0" fill="rgb(252,149,25)" rx="2" ry="2" />
<text  x="77.39" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).ptraceSyscallExit (54 samples, 0.02%)</title><rect x="220.6" y="805" width="0.2" height="15.0" fill="rgb(205,45,22)" rx="2" ry="2" />
<text  x="223.60" y="815.5" ></text>
</g>
<g >
<title>runtime.(*mheap).allocSpan (42 samples, 0.01%)</title><rect x="1153.1" y="693" width="0.2" height="15.0" fill="rgb(211,6,41)" rx="2" ry="2" />
<text  x="1156.11" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).Broadcast (255 samples, 0.08%)</title><rect x="785.1" y="645" width="0.9" height="15.0" fill="rgb(211,9,15)" rx="2" ry="2" />
<text  x="788.08" y="655.5" ></text>
</g>
<g >
<title>runtime.chansend.func1 (29 samples, 0.01%)</title><rect x="1003.4" y="725" width="0.1" height="15.0" fill="rgb(223,100,7)" rx="2" ry="2" />
<text  x="1006.37" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (357 samples, 0.11%)</title><rect x="196.5" y="245" width="1.3" height="15.0" fill="rgb(243,144,24)" rx="2" ry="2" />
<text  x="199.53" y="255.5" ></text>
</g>
<g >
<title>runtime.pcdatavalue (125 samples, 0.04%)</title><rect x="1103.3" y="677" width="0.5" height="15.0" fill="rgb(207,204,37)" rx="2" ry="2" />
<text  x="1106.32" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (93 samples, 0.03%)</title><rect x="1060.5" y="597" width="0.3" height="15.0" fill="rgb(213,188,10)" rx="2" ry="2" />
<text  x="1063.47" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (440 samples, 0.13%)</title><rect x="1063.2" y="677" width="1.6" height="15.0" fill="rgb(235,95,44)" rx="2" ry="2" />
<text  x="1066.21" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FSContext).RootDirectory (71 samples, 0.02%)</title><rect x="189.6" y="741" width="0.3" height="15.0" fill="rgb(207,120,32)" rx="2" ry="2" />
<text  x="192.62" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).get (98 samples, 0.03%)</title><rect x="1029.3" y="645" width="0.3" height="15.0" fill="rgb(254,154,6)" rx="2" ry="2" />
<text  x="1032.27" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBuffer).Views (290 samples, 0.09%)</title><rect x="926.4" y="805" width="1.1" height="15.0" fill="rgb(241,190,44)" rx="2" ry="2" />
<text  x="929.44" y="815.5" ></text>
</g>
<g >
<title>runtime.dodeltimer0 (34 samples, 0.01%)</title><rect x="903.9" y="693" width="0.1" height="15.0" fill="rgb(211,6,8)" rx="2" ry="2" />
<text  x="906.86" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*channelCallback).Callback (88 samples, 0.03%)</title><rect x="1071.5" y="709" width="0.3" height="15.0" fill="rgb(234,32,21)" rx="2" ry="2" />
<text  x="1074.53" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (18,643 samples, 5.71%)</title><rect x="930.7" y="789" width="67.4" height="15.0" fill="rgb(239,122,19)" rx="2" ry="2" />
<text  x="933.71" y="799.5" >[[kerne..</text>
</g>
<g >
<title>runtime.scavengeSleep (530 samples, 0.16%)</title><rect x="1107.5" y="853" width="1.9" height="15.0" fill="rgb(218,8,48)" rx="2" ry="2" />
<text  x="1110.49" y="863.5" ></text>
</g>
<g >
<title>runtime.mapassign (124 samples, 0.04%)</title><rect x="1025.7" y="773" width="0.5" height="15.0" fill="rgb(239,228,44)" rx="2" ry="2" />
<text  x="1028.74" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (292 samples, 0.09%)</title><rect x="1188.7" y="661" width="1.0" height="15.0" fill="rgb(254,4,50)" rx="2" ry="2" />
<text  x="1191.65" y="671.5" ></text>
</g>
<g >
<title>runtime.adjustframe (481 samples, 0.15%)</title><rect x="1014.8" y="725" width="1.7" height="15.0" fill="rgb(230,178,24)" rx="2" ry="2" />
<text  x="1017.80" y="735.5" ></text>
</g>
<g >
<title>runtime.gentraceback (46 samples, 0.01%)</title><rect x="767.9" y="517" width="0.2" height="15.0" fill="rgb(228,66,30)" rx="2" ry="2" />
<text  x="770.93" y="527.5" ></text>
</g>
<g >
<title>runtime.newobject (90 samples, 0.03%)</title><rect x="72.5" y="725" width="0.3" height="15.0" fill="rgb(214,139,52)" rx="2" ry="2" />
<text  x="75.45" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (857 samples, 0.26%)</title><rect x="124.3" y="517" width="3.1" height="15.0" fill="rgb(253,161,3)" rx="2" ry="2" />
<text  x="127.27" y="527.5" ></text>
</g>
<g >
<title>runtime.funcname (28 samples, 0.01%)</title><rect x="1005.6" y="773" width="0.1" height="15.0" fill="rgb(243,41,50)" rx="2" ry="2" />
<text  x="1008.60" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).acquirePrimaryAddressRLocked (307 samples, 0.09%)</title><rect x="725.5" y="693" width="1.1" height="15.0" fill="rgb(230,54,37)" rx="2" ry="2" />
<text  x="728.46" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (12,903 samples, 3.95%)</title><rect x="950.0" y="549" width="46.7" height="15.0" fill="rgb(251,214,19)" rx="2" ry="2" />
<text  x="953.04" y="559.5" >[[ke..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).IsBroadcast (83 samples, 0.03%)</title><rect x="1036.4" y="661" width="0.3" height="15.0" fill="rgb(236,157,18)" rx="2" ry="2" />
<text  x="1039.41" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).AcquireAssignedAddressOrMatching (1,447 samples, 0.44%)</title><rect x="733.4" y="693" width="5.2" height="15.0" fill="rgb(212,144,24)" rx="2" ry="2" />
<text  x="736.35" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*overlayFileOperations).Write (47 samples, 0.01%)</title><rect x="21.6" y="789" width="0.1" height="15.0" fill="rgb(243,161,48)" rx="2" ry="2" />
<text  x="24.57" y="799.5" ></text>
</g>
<g >
<title>runtime.notesleep (166 samples, 0.05%)</title><rect x="22.7" y="789" width="0.6" height="15.0" fill="rgb(229,51,24)" rx="2" ry="2" />
<text  x="25.68" y="799.5" ></text>
</g>
<g >
<title>runtime.adjustpointers (66 samples, 0.02%)</title><rect x="1014.9" y="709" width="0.3" height="15.0" fill="rgb(226,37,20)" rx="2" ry="2" />
<text  x="1017.95" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).DeliverOutboundPacket (29 samples, 0.01%)</title><rect x="1091.4" y="565" width="0.1" height="15.0" fill="rgb(214,153,30)" rx="2" ry="2" />
<text  x="1094.43" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="1185.8" y="757" width="0.2" height="15.0" fill="rgb(224,161,2)" rx="2" ry="2" />
<text  x="1188.84" y="767.5" ></text>
</g>
<g >
<title>runtime.selectnbsend (579 samples, 0.18%)</title><rect x="1002.0" y="773" width="2.0" height="15.0" fill="rgb(241,20,37)" rx="2" ry="2" />
<text  x="1004.95" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (37 samples, 0.01%)</title><rect x="1151.1" y="677" width="0.1" height="15.0" fill="rgb(218,51,20)" rx="2" ry="2" />
<text  x="1154.10" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (392 samples, 0.12%)</title><rect x="705.3" y="581" width="1.4" height="15.0" fill="rgb(227,131,20)" rx="2" ry="2" />
<text  x="708.28" y="591.5" ></text>
</g>
<g >
<title>runtime.casgstatus (144 samples, 0.04%)</title><rect x="97.6" y="629" width="0.5" height="15.0" fill="rgb(250,15,12)" rx="2" ry="2" />
<text  x="100.58" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (805 samples, 0.25%)</title><rect x="755.2" y="357" width="2.9" height="15.0" fill="rgb(237,73,51)" rx="2" ry="2" />
<text  x="758.16" y="367.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,180 samples, 1.59%)</title><rect x="973.9" y="261" width="18.7" height="15.0" fill="rgb(235,85,54)" rx="2" ry="2" />
<text  x="976.86" y="271.5" ></text>
</g>
<g >
<title>runtime.memmove (48 samples, 0.01%)</title><rect x="717.4" y="805" width="0.1" height="15.0" fill="rgb(207,128,54)" rx="2" ry="2" />
<text  x="720.38" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).WritePacket (1,214 samples, 0.37%)</title><rect x="209.9" y="533" width="4.4" height="15.0" fill="rgb(219,48,35)" rx="2" ry="2" />
<text  x="212.88" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).SleepStart (652 samples, 0.20%)</title><rect x="83.0" y="709" width="2.3" height="15.0" fill="rgb(251,203,42)" rx="2" ry="2" />
<text  x="85.95" y="719.5" ></text>
</g>
<g >
<title>runtime.notewakeup (363 samples, 0.11%)</title><rect x="1065.2" y="709" width="1.3" height="15.0" fill="rgb(217,195,20)" rx="2" ry="2" />
<text  x="1068.21" y="719.5" ></text>
</g>
<g >
<title>runtime.closechan (145 samples, 0.04%)</title><rect x="1050.9" y="741" width="0.5" height="15.0" fill="rgb(208,129,44)" rx="2" ry="2" />
<text  x="1053.87" y="751.5" ></text>
</g>
<g >
<title>runtime.notewakeup (540 samples, 0.17%)</title><rect x="1187.8" y="773" width="1.9" height="15.0" fill="rgb(223,92,35)" rx="2" ry="2" />
<text  x="1190.76" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendSegment (2,166 samples, 0.66%)</title><rect x="208.1" y="661" width="7.8" height="15.0" fill="rgb(206,153,29)" rx="2" ry="2" />
<text  x="211.06" y="671.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (15,338 samples, 4.70%)</title><rect x="640.1" y="821" width="55.5" height="15.0" fill="rgb(236,2,43)" rx="2" ry="2" />
<text  x="643.14" y="831.5" >sysca..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (142 samples, 0.04%)</title><rect x="701.1" y="613" width="0.5" height="15.0" fill="rgb(230,70,33)" rx="2" ry="2" />
<text  x="704.08" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="905.4" y="677" width="0.1" height="15.0" fill="rgb(244,209,39)" rx="2" ry="2" />
<text  x="908.40" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/rand.(*bufferedReader).Read (41 samples, 0.01%)</title><rect x="1041.3" y="725" width="0.1" height="15.0" fill="rgb(251,97,44)" rx="2" ry="2" />
<text  x="1044.27" y="735.5" ></text>
</g>
<g >
<title>runtime.lock2 (46 samples, 0.01%)</title><rect x="904.9" y="741" width="0.2" height="15.0" fill="rgb(220,78,8)" rx="2" ry="2" />
<text  x="907.91" y="751.5" ></text>
</g>
<g >
<title>runtime.schedule (2,709 samples, 0.83%)</title><rect x="1057.0" y="773" width="9.8" height="15.0" fill="rgb(205,3,54)" rx="2" ry="2" />
<text  x="1060.00" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (62 samples, 0.02%)</title><rect x="452.3" y="677" width="0.2" height="15.0" fill="rgb(205,24,37)" rx="2" ry="2" />
<text  x="455.28" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*endpoint).WritePackets (30 samples, 0.01%)</title><rect x="895.8" y="853" width="0.1" height="15.0" fill="rgb(209,43,29)" rx="2" ry="2" />
<text  x="898.75" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).initHardwareGSO (161 samples, 0.05%)</title><rect x="1038.2" y="773" width="0.6" height="15.0" fill="rgb(242,120,17)" rx="2" ry="2" />
<text  x="1041.20" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (41 samples, 0.01%)</title><rect x="701.8" y="613" width="0.2" height="15.0" fill="rgb(223,1,5)" rx="2" ry="2" />
<text  x="704.85" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (43 samples, 0.01%)</title><rect x="776.9" y="645" width="0.1" height="15.0" fill="rgb(247,64,21)" rx="2" ry="2" />
<text  x="779.85" y="655.5" ></text>
</g>
<g >
<title>runtime.makemap_small (37 samples, 0.01%)</title><rect x="1023.5" y="773" width="0.1" height="15.0" fill="rgb(253,16,22)" rx="2" ry="2" />
<text  x="1026.47" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).Subnet (36 samples, 0.01%)</title><rect x="788.4" y="677" width="0.1" height="15.0" fill="rgb(205,204,43)" rx="2" ry="2" />
<text  x="791.36" y="687.5" ></text>
</g>
<g >
<title>runtime.morestack (1,302 samples, 0.40%)</title><rect x="1094.5" y="789" width="4.8" height="15.0" fill="rgb(223,29,50)" rx="2" ry="2" />
<text  x="1097.55" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (383 samples, 0.12%)</title><rect x="196.4" y="261" width="1.4" height="15.0" fill="rgb(214,216,14)" rx="2" ry="2" />
<text  x="199.43" y="271.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Read (526 samples, 0.16%)</title><rect x="175.8" y="709" width="1.9" height="15.0" fill="rgb(207,0,31)" rx="2" ry="2" />
<text  x="178.78" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (14,096 samples, 4.32%)</title><rect x="945.7" y="629" width="51.0" height="15.0" fill="rgb(243,30,17)" rx="2" ry="2" />
<text  x="948.73" y="639.5" >[[ker..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (299 samples, 0.09%)</title><rect x="1125.7" y="613" width="1.1" height="15.0" fill="rgb(233,66,19)" rx="2" ry="2" />
<text  x="1128.71" y="623.5" ></text>
</g>
<g >
<title>runtime.runqgrab (333 samples, 0.10%)</title><rect x="700.4" y="757" width="1.2" height="15.0" fill="rgb(213,114,36)" rx="2" ry="2" />
<text  x="703.39" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.overlayCheck (253 samples, 0.08%)</title><rect x="188.0" y="677" width="0.9" height="15.0" fill="rgb(243,162,27)" rx="2" ry="2" />
<text  x="191.03" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (81 samples, 0.02%)</title><rect x="13.7" y="661" width="0.3" height="15.0" fill="rgb(246,178,1)" rx="2" ry="2" />
<text  x="16.69" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (45 samples, 0.01%)</title><rect x="192.1" y="645" width="0.2" height="15.0" fill="rgb(214,91,9)" rx="2" ry="2" />
<text  x="195.10" y="655.5" ></text>
</g>
<g >
<title>runtime.mapaccess1_fast64 (55 samples, 0.02%)</title><rect x="21.6" y="869" width="0.2" height="15.0" fill="rgb(250,213,54)" rx="2" ry="2" />
<text  x="24.56" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (118 samples, 0.04%)</title><rect x="157.4" y="469" width="0.4" height="15.0" fill="rgb(249,155,50)" rx="2" ry="2" />
<text  x="160.39" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).closeNoShutdownLocked (320 samples, 0.10%)</title><rect x="65.0" y="677" width="1.1" height="15.0" fill="rgb(240,145,38)" rx="2" ry="2" />
<text  x="67.97" y="687.5" ></text>
</g>
<g >
<title>runtime.notesleep (147 samples, 0.05%)</title><rect x="1108.5" y="757" width="0.5" height="15.0" fill="rgb(208,146,35)" rx="2" ry="2" />
<text  x="1111.51" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).Promiscuous (55 samples, 0.02%)</title><rect x="775.6" y="709" width="0.2" height="15.0" fill="rgb(214,8,44)" rx="2" ry="2" />
<text  x="778.59" y="719.5" ></text>
</g>
<g >
<title>runtime.chanrecv (430 samples, 0.13%)</title><rect x="705.2" y="837" width="1.5" height="15.0" fill="rgb(218,212,39)" rx="2" ry="2" />
<text  x="708.15" y="847.5" ></text>
</g>
<g >
<title>runtime.growWork (57 samples, 0.02%)</title><rect x="1023.9" y="757" width="0.2" height="15.0" fill="rgb(244,210,14)" rx="2" ry="2" />
<text  x="1026.93" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*iovecBuffer).nextIovecs (675 samples, 0.21%)</title><rect x="709.8" y="821" width="2.4" height="15.0" fill="rgb(211,27,53)" rx="2" ry="2" />
<text  x="712.79" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (697 samples, 0.21%)</title><rect x="919.4" y="677" width="2.5" height="15.0" fill="rgb(205,28,1)" rx="2" ry="2" />
<text  x="922.35" y="687.5" ></text>
</g>
<g >
<title>runtime.adjustpointers (97 samples, 0.03%)</title><rect x="1095.4" y="709" width="0.3" height="15.0" fill="rgb(228,66,18)" rx="2" ry="2" />
<text  x="1098.40" y="719.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (574 samples, 0.18%)</title><rect x="765.3" y="565" width="2.0" height="15.0" fill="rgb(221,85,50)" rx="2" ry="2" />
<text  x="768.26" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (105 samples, 0.03%)</title><rect x="1108.7" y="693" width="0.3" height="15.0" fill="rgb(252,12,34)" rx="2" ry="2" />
<text  x="1111.66" y="703.5" ></text>
</g>
<g >
<title>runtime.usleep (43 samples, 0.01%)</title><rect x="1185.8" y="773" width="0.2" height="15.0" fill="rgb(240,64,18)" rx="2" ry="2" />
<text  x="1188.84" y="783.5" ></text>
</g>
<g >
<title>runtime.startlockedm (71 samples, 0.02%)</title><rect x="1189.7" y="821" width="0.3" height="15.0" fill="rgb(212,166,30)" rx="2" ry="2" />
<text  x="1192.74" y="831.5" ></text>
</g>
<g >
<title>runtime.growWork (82 samples, 0.03%)</title><rect x="1025.9" y="757" width="0.2" height="15.0" fill="rgb(223,9,41)" rx="2" ry="2" />
<text  x="1028.85" y="767.5" ></text>
</g>
<g >
<title>runtime.gfget (40 samples, 0.01%)</title><rect x="1080.4" y="597" width="0.1" height="15.0" fill="rgb(234,49,10)" rx="2" ry="2" />
<text  x="1083.36" y="607.5" ></text>
</g>
<g >
<title>runtime.(*mheap).allocSpan (41 samples, 0.01%)</title><rect x="711.5" y="677" width="0.1" height="15.0" fill="rgb(210,159,28)" rx="2" ry="2" />
<text  x="714.47" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).DeliverTransportPacket (9,970 samples, 3.06%)</title><rect x="739.5" y="709" width="36.1" height="15.0" fill="rgb(249,227,39)" rx="2" ry="2" />
<text  x="742.50" y="719.5" >gvi..</text>
</g>
<g >
<title>runtime.pcvalue (424 samples, 0.13%)</title><rect x="1135.8" y="741" width="1.5" height="15.0" fill="rgb(243,130,27)" rx="2" ry="2" />
<text  x="1138.80" y="751.5" ></text>
</g>
<g >
<title>runtime.addtimer (30 samples, 0.01%)</title><rect x="1019.7" y="789" width="0.1" height="15.0" fill="rgb(214,179,0)" rx="2" ry="2" />
<text  x="1022.72" y="799.5" ></text>
</g>
<g >
<title>runtime.memmove (29 samples, 0.01%)</title><rect x="1105.0" y="741" width="0.1" height="15.0" fill="rgb(238,15,23)" rx="2" ry="2" />
<text  x="1107.97" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (242 samples, 0.07%)</title><rect x="704.0" y="645" width="0.9" height="15.0" fill="rgb(247,115,15)" rx="2" ry="2" />
<text  x="707.00" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (60 samples, 0.02%)</title><rect x="793.3" y="725" width="0.2" height="15.0" fill="rgb(205,104,1)" rx="2" ry="2" />
<text  x="796.29" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).IsBroadcast (306 samples, 0.09%)</title><rect x="785.0" y="661" width="1.1" height="15.0" fill="rgb(225,183,11)" rx="2" ry="2" />
<text  x="787.97" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (40 samples, 0.01%)</title><rect x="21.6" y="549" width="0.1" height="15.0" fill="rgb(216,203,5)" rx="2" ry="2" />
<text  x="24.59" y="559.5" ></text>
</g>
<g >
<title>runtime.send (32 samples, 0.01%)</title><rect x="1071.7" y="661" width="0.1" height="15.0" fill="rgb(250,157,52)" rx="2" ry="2" />
<text  x="1074.72" y="671.5" ></text>
</g>
<g >
<title>runtime.makeslice (51 samples, 0.02%)</title><rect x="204.4" y="725" width="0.1" height="15.0" fill="rgb(253,18,10)" rx="2" ry="2" />
<text  x="207.36" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,976 samples, 0.91%)</title><rect x="884.1" y="661" width="10.8" height="15.0" fill="rgb(224,34,21)" rx="2" ry="2" />
<text  x="887.12" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (31 samples, 0.01%)</title><rect x="1109.2" y="629" width="0.1" height="15.0" fill="rgb(219,194,18)" rx="2" ry="2" />
<text  x="1112.19" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).assertTaskGoroutine (28 samples, 0.01%)</title><rect x="85.6" y="709" width="0.1" height="15.0" fill="rgb(209,221,45)" rx="2" ry="2" />
<text  x="88.56" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (95 samples, 0.03%)</title><rect x="149.0" y="597" width="0.4" height="15.0" fill="rgb(221,32,48)" rx="2" ry="2" />
<text  x="152.01" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (57 samples, 0.02%)</title><rect x="57.2" y="661" width="0.2" height="15.0" fill="rgb(243,64,17)" rx="2" ry="2" />
<text  x="60.20" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (100 samples, 0.03%)</title><rect x="1062.1" y="677" width="0.4" height="15.0" fill="rgb(208,190,5)" rx="2" ry="2" />
<text  x="1065.14" y="687.5" ></text>
</g>
<g >
<title>runtime.futex (141 samples, 0.04%)</title><rect x="12.9" y="725" width="0.5" height="15.0" fill="rgb(211,40,20)" rx="2" ry="2" />
<text  x="15.88" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (325 samples, 0.10%)</title><rect x="988.5" y="117" width="1.2" height="15.0" fill="rgb(218,198,8)" rx="2" ry="2" />
<text  x="991.51" y="127.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (67 samples, 0.02%)</title><rect x="18.8" y="837" width="0.2" height="15.0" fill="rgb(228,130,24)" rx="2" ry="2" />
<text  x="21.79" y="847.5" ></text>
</g>
<g >
<title>runtime.freeStackSpans (42 samples, 0.01%)</title><rect x="1111.0" y="821" width="0.2" height="15.0" fill="rgb(208,209,27)" rx="2" ry="2" />
<text  x="1114.04" y="831.5" ></text>
</g>
<g >
<title>runtime.wakep (128 samples, 0.04%)</title><rect x="18.2" y="789" width="0.5" height="15.0" fill="rgb(214,72,14)" rx="2" ry="2" />
<text  x="21.21" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (869 samples, 0.27%)</title><rect x="1123.7" y="757" width="3.1" height="15.0" fill="rgb(246,112,24)" rx="2" ry="2" />
<text  x="1126.65" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/binary.Marshal (492 samples, 0.15%)</title><rect x="924.1" y="805" width="1.8" height="15.0" fill="rgb(239,195,35)" rx="2" ry="2" />
<text  x="927.13" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (40 samples, 0.01%)</title><rect x="200.9" y="261" width="0.2" height="15.0" fill="rgb(205,206,40)" rx="2" ry="2" />
<text  x="203.92" y="271.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/packetsocket.(*endpoint).WritePacket (171 samples, 0.05%)</title><rect x="1073.8" y="581" width="0.6" height="15.0" fill="rgb(226,58,42)" rx="2" ry="2" />
<text  x="1076.76" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="149.2" y="437" width="0.2" height="15.0" fill="rgb(229,47,40)" rx="2" ry="2" />
<text  x="152.22" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).DecRef (1,479 samples, 0.45%)</title><rect x="63.4" y="757" width="5.4" height="15.0" fill="rgb(235,202,7)" rx="2" ry="2" />
<text  x="66.43" y="767.5" ></text>
</g>
<g >
<title>ext4_inode_csum.isra.62 (40 samples, 0.01%)</title><rect x="200.5" y="197" width="0.2" height="15.0" fill="rgb(238,127,6)" rx="2" ry="2" />
<text  x="203.55" y="207.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,735 samples, 0.84%)</title><rect x="748.2" y="485" width="9.9" height="15.0" fill="rgb(246,70,7)" rx="2" ry="2" />
<text  x="751.18" y="495.5" ></text>
</g>
<g >
<title>runtime.findObject (1,335 samples, 0.41%)</title><rect x="1173.2" y="805" width="4.8" height="15.0" fill="rgb(243,210,49)" rx="2" ry="2" />
<text  x="1176.21" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*AtomicRefCount).DecRefWithDestructor (103 samples, 0.03%)</title><rect x="64.0" y="629" width="0.4" height="15.0" fill="rgb(223,108,26)" rx="2" ry="2" />
<text  x="66.98" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (114 samples, 0.03%)</title><rect x="734.4" y="661" width="0.4" height="15.0" fill="rgb(233,83,3)" rx="2" ry="2" />
<text  x="737.43" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (199 samples, 0.06%)</title><rect x="1130.2" y="757" width="0.7" height="15.0" fill="rgb(222,134,48)" rx="2" ry="2" />
<text  x="1133.18" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isResolutionRequiredRLocked (30 samples, 0.01%)</title><rect x="1092.1" y="661" width="0.1" height="15.0" fill="rgb(224,59,48)" rx="2" ry="2" />
<text  x="1095.14" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (172 samples, 0.05%)</title><rect x="904.3" y="725" width="0.6" height="15.0" fill="rgb(214,24,47)" rx="2" ry="2" />
<text  x="907.28" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (30 samples, 0.01%)</title><rect x="65.9" y="437" width="0.1" height="15.0" fill="rgb(207,129,49)" rx="2" ry="2" />
<text  x="68.86" y="447.5" ></text>
</g>
<g >
<title>memeqbody (33 samples, 0.01%)</title><rect x="735.7" y="661" width="0.2" height="15.0" fill="rgb(235,188,17)" rx="2" ry="2" />
<text  x="738.75" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (50 samples, 0.02%)</title><rect x="1120.2" y="741" width="0.2" height="15.0" fill="rgb(218,37,21)" rx="2" ry="2" />
<text  x="1123.20" y="751.5" ></text>
</g>
<g >
<title>syscall.Wait4 (61,115 samples, 18.73%)</title><rect x="230.1" y="789" width="221.0" height="15.0" fill="rgb(217,138,23)" rx="2" ry="2" />
<text  x="233.06" y="799.5" >syscall.Wait4</text>
</g>
<g >
<title>[[kernel.kallsyms]] (57 samples, 0.02%)</title><rect x="452.3" y="629" width="0.2" height="15.0" fill="rgb(253,123,21)" rx="2" ry="2" />
<text  x="455.30" y="639.5" ></text>
</g>
<g >
<title>runtime.pcvalue (200 samples, 0.06%)</title><rect x="1015.8" y="677" width="0.7" height="15.0" fill="rgb(218,156,12)" rx="2" ry="2" />
<text  x="1018.76" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="700.0" y="661" width="0.1" height="15.0" fill="rgb(237,129,32)" rx="2" ry="2" />
<text  x="702.98" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (147 samples, 0.05%)</title><rect x="451.7" y="501" width="0.5" height="15.0" fill="rgb(214,210,4)" rx="2" ry="2" />
<text  x="454.70" y="511.5" ></text>
</g>
<g >
<title>runtime.dodeltimer (60 samples, 0.02%)</title><rect x="96.9" y="613" width="0.2" height="15.0" fill="rgb(209,113,34)" rx="2" ry="2" />
<text  x="99.92" y="623.5" ></text>
</g>
<g >
<title>runtime.step (54 samples, 0.02%)</title><rect x="1103.6" y="645" width="0.1" height="15.0" fill="rgb(246,94,23)" rx="2" ry="2" />
<text  x="1106.55" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).GetKind (78 samples, 0.02%)</title><rect x="725.6" y="661" width="0.3" height="15.0" fill="rgb(244,111,31)" rx="2" ry="2" />
<text  x="728.60" y="671.5" ></text>
</g>
<g >
<title>runtime.adjustframe (51 samples, 0.02%)</title><rect x="1013.9" y="709" width="0.2" height="15.0" fill="rgb(228,3,50)" rx="2" ry="2" />
<text  x="1016.94" y="719.5" ></text>
</g>
<g >
<title>runtime.schedule (79 samples, 0.02%)</title><rect x="1111.3" y="821" width="0.3" height="15.0" fill="rgb(221,82,29)" rx="2" ry="2" />
<text  x="1114.27" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).HandlePacket (69 samples, 0.02%)</title><rect x="719.9" y="757" width="0.2" height="15.0" fill="rgb(240,17,37)" rx="2" ry="2" />
<text  x="722.86" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (49,441 samples, 15.15%)</title><rect x="459.2" y="789" width="178.8" height="15.0" fill="rgb(220,154,10)" rx="2" ry="2" />
<text  x="462.22" y="799.5" >[[kernel.kallsyms]]</text>
</g>
<g >
<title>runtime.siftdownTimer (54 samples, 0.02%)</title><rect x="1082.0" y="565" width="0.2" height="15.0" fill="rgb(231,168,2)" rx="2" ry="2" />
<text  x="1084.96" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.Copy (28 samples, 0.01%)</title><rect x="74.7" y="677" width="0.1" height="15.0" fill="rgb(242,86,13)" rx="2" ry="2" />
<text  x="77.66" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (61 samples, 0.02%)</title><rect x="1060.6" y="565" width="0.2" height="15.0" fill="rgb(237,188,29)" rx="2" ry="2" />
<text  x="1063.58" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (760 samples, 0.23%)</title><rect x="210.4" y="405" width="2.8" height="15.0" fill="rgb(243,118,53)" rx="2" ry="2" />
<text  x="213.43" y="415.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).Writev (3,517 samples, 1.08%)</title><rect x="190.6" y="741" width="12.7" height="15.0" fill="rgb(233,10,5)" rx="2" ry="2" />
<text  x="193.56" y="751.5" ></text>
</g>
<g >
<title>runtime.schedule (44 samples, 0.01%)</title><rect x="696.8" y="805" width="0.2" height="15.0" fill="rgb(227,128,24)" rx="2" ry="2" />
<text  x="699.81" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/rawfile.BlockingRecvMMsg (89 samples, 0.03%)</title><rect x="822.2" y="821" width="0.3" height="15.0" fill="rgb(213,192,38)" rx="2" ry="2" />
<text  x="825.17" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (57 samples, 0.02%)</title><rect x="163.3" y="709" width="0.2" height="15.0" fill="rgb(248,98,21)" rx="2" ry="2" />
<text  x="166.26" y="719.5" ></text>
</g>
<g >
<title>runtime.mallocgc (64 samples, 0.02%)</title><rect x="1068.6" y="821" width="0.2" height="15.0" fill="rgb(226,186,23)" rx="2" ry="2" />
<text  x="1071.59" y="831.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc.func1 (49 samples, 0.02%)</title><rect x="1106.6" y="805" width="0.2" height="15.0" fill="rgb(217,73,36)" rx="2" ry="2" />
<text  x="1109.62" y="815.5" ></text>
</g>
<g >
<title>runtime.gentraceback (848 samples, 0.26%)</title><rect x="1147.6" y="741" width="3.1" height="15.0" fill="rgb(239,109,17)" rx="2" ry="2" />
<text  x="1150.59" y="751.5" ></text>
</g>
<g >
<title>runtime.wakep (564 samples, 0.17%)</title><rect x="1187.7" y="805" width="2.0" height="15.0" fill="rgb(210,31,44)" rx="2" ry="2" />
<text  x="1190.69" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (8,017 samples, 2.46%)</title><rect x="666.6" y="709" width="29.0" height="15.0" fill="rgb(253,188,51)" rx="2" ry="2" />
<text  x="669.62" y="719.5" >[[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.EpollWait (51 samples, 0.02%)</title><rect x="219.4" y="789" width="0.2" height="15.0" fill="rgb(244,181,52)" rx="2" ry="2" />
<text  x="222.41" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (337 samples, 0.10%)</title><rect x="1121.9" y="725" width="1.2" height="15.0" fill="rgb(213,123,2)" rx="2" ry="2" />
<text  x="1124.93" y="735.5" ></text>
</g>
<g >
<title>runtime.findrunnable (67 samples, 0.02%)</title><rect x="17.2" y="805" width="0.2" height="15.0" fill="rgb(235,22,25)" rx="2" ry="2" />
<text  x="20.19" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).rcvWndScaleForHandshake (43 samples, 0.01%)</title><rect x="1038.8" y="773" width="0.2" height="15.0" fill="rgb(232,210,46)" rx="2" ry="2" />
<text  x="1041.84" y="783.5" ></text>
</g>
<g >
<title>runtime.tracebackdefers (120 samples, 0.04%)</title><rect x="1013.8" y="725" width="0.5" height="15.0" fill="rgb(211,79,52)" rx="2" ry="2" />
<text  x="1016.84" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/interrupt.(*Forwarder).Disable (52 samples, 0.02%)</title><rect x="223.7" y="821" width="0.2" height="15.0" fill="rgb(208,41,6)" rx="2" ry="2" />
<text  x="226.72" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (96 samples, 0.03%)</title><rect x="1152.8" y="661" width="0.3" height="15.0" fill="rgb(226,156,34)" rx="2" ry="2" />
<text  x="1155.76" y="671.5" ></text>
</g>
<g >
<title>runtime.greyobject (62 samples, 0.02%)</title><rect x="1132.3" y="821" width="0.3" height="15.0" fill="rgb(243,185,26)" rx="2" ry="2" />
<text  x="1135.34" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBuffer).consume (72 samples, 0.02%)</title><rect x="793.7" y="709" width="0.3" height="15.0" fill="rgb(207,48,30)" rx="2" ry="2" />
<text  x="796.74" y="719.5" ></text>
</g>
<g >
<title>runtime.schedule (325 samples, 0.10%)</title><rect x="17.5" y="821" width="1.2" height="15.0" fill="rgb(217,119,53)" rx="2" ry="2" />
<text  x="20.50" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).protocolListenLoop (6,472 samples, 1.98%)</title><rect x="1021.3" y="869" width="23.4" height="15.0" fill="rgb(238,25,16)" rx="2" ry="2" />
<text  x="1024.31" y="879.5" >g..</text>
</g>
<g >
<title>runtime.newobject (49 samples, 0.02%)</title><rect x="1010.7" y="741" width="0.1" height="15.0" fill="rgb(254,182,31)" rx="2" ry="2" />
<text  x="1013.65" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="1129.6" y="741" width="0.1" height="15.0" fill="rgb(210,34,36)" rx="2" ry="2" />
<text  x="1132.55" y="751.5" ></text>
</g>
<g >
<title>runtime.park_m (18,393 samples, 5.64%)</title><rect x="92.0" y="677" width="66.5" height="15.0" fill="rgb(212,133,21)" rx="2" ry="2" />
<text  x="95.01" y="687.5" >runtime..</text>
</g>
<g >
<title>runtime.systemstack (3,300 samples, 1.01%)</title><rect x="746.4" y="597" width="11.9" height="15.0" fill="rgb(249,55,18)" rx="2" ry="2" />
<text  x="749.36" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.overlayGetlink (105 samples, 0.03%)</title><rect x="189.1" y="677" width="0.4" height="15.0" fill="rgb(240,69,37)" rx="2" ry="2" />
<text  x="192.07" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborEntry).cancelJobLocked (474 samples, 0.15%)</title><rect x="1084.7" y="725" width="1.7" height="15.0" fill="rgb(246,186,36)" rx="2" ry="2" />
<text  x="1087.68" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (343 samples, 0.11%)</title><rect x="1127.3" y="741" width="1.3" height="15.0" fill="rgb(206,149,17)" rx="2" ry="2" />
<text  x="1130.32" y="751.5" ></text>
</g>
<g >
<title>runtime.growslice (42 samples, 0.01%)</title><rect x="46.9" y="693" width="0.1" height="15.0" fill="rgb(211,8,32)" rx="2" ry="2" />
<text  x="49.88" y="703.5" ></text>
</g>
<g >
<title>runtime.park_m (35 samples, 0.01%)</title><rect x="1021.6" y="805" width="0.1" height="15.0" fill="rgb(249,176,8)" rx="2" ry="2" />
<text  x="1024.62" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (471 samples, 0.14%)</title><rect x="1121.4" y="757" width="1.7" height="15.0" fill="rgb(213,61,45)" rx="2" ry="2" />
<text  x="1124.44" y="767.5" ></text>
</g>
<g >
<title>nf_nat_packet (37 samples, 0.01%)</title><rect x="965.0" y="453" width="0.1" height="15.0" fill="rgb(211,134,1)" rx="2" ry="2" />
<text  x="967.99" y="463.5" ></text>
</g>
<g >
<title>sync.(*Pool).Get (96 samples, 0.03%)</title><rect x="45.7" y="693" width="0.4" height="15.0" fill="rgb(245,59,39)" rx="2" ry="2" />
<text  x="48.75" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (63 samples, 0.02%)</title><rect x="701.8" y="709" width="0.2" height="15.0" fill="rgb(205,20,54)" rx="2" ry="2" />
<text  x="704.77" y="719.5" ></text>
</g>
<g >
<title>time.startTimer (66 samples, 0.02%)</title><rect x="1040.9" y="741" width="0.2" height="15.0" fill="rgb(223,135,17)" rx="2" ry="2" />
<text  x="1043.87" y="751.5" ></text>
</g>
<g >
<title>runtime.assertE2I (35 samples, 0.01%)</title><rect x="67.4" y="661" width="0.1" height="15.0" fill="rgb(217,175,19)" rx="2" ry="2" />
<text  x="70.35" y="671.5" ></text>
</g>
<g >
<title>runtime.mallocgc (53 samples, 0.02%)</title><rect x="927.7" y="789" width="0.2" height="15.0" fill="rgb(210,51,39)" rx="2" ry="2" />
<text  x="930.66" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (293 samples, 0.09%)</title><rect x="1063.7" y="597" width="1.1" height="15.0" fill="rgb(214,63,0)" rx="2" ry="2" />
<text  x="1066.74" y="607.5" ></text>
</g>
<g >
<title>runtime.unlock2 (44 samples, 0.01%)</title><rect x="1126.9" y="805" width="0.1" height="15.0" fill="rgb(212,6,31)" rx="2" ry="2" />
<text  x="1129.85" y="815.5" ></text>
</g>
<g >
<title>runtime.mapiternext (164 samples, 0.05%)</title><rect x="789.5" y="661" width="0.6" height="15.0" fill="rgb(226,187,50)" rx="2" ry="2" />
<text  x="792.50" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).ptraceSyscallExit (70 samples, 0.02%)</title><rect x="217.3" y="789" width="0.3" height="15.0" fill="rgb(227,20,13)" rx="2" ry="2" />
<text  x="220.32" y="799.5" ></text>
</g>
<g >
<title>runtime.newproc1 (108 samples, 0.03%)</title><rect x="1005.3" y="805" width="0.4" height="15.0" fill="rgb(242,153,2)" rx="2" ry="2" />
<text  x="1008.34" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (108 samples, 0.03%)</title><rect x="115.5" y="469" width="0.4" height="15.0" fill="rgb(248,56,1)" rx="2" ry="2" />
<text  x="118.49" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).startRead (85 samples, 0.03%)</title><rect x="177.3" y="693" width="0.3" height="15.0" fill="rgb(210,228,48)" rx="2" ry="2" />
<text  x="180.25" y="703.5" ></text>
</g>
<g >
<title>runtime.funcspdelta (163 samples, 0.05%)</title><rect x="1150.1" y="725" width="0.6" height="15.0" fill="rgb(228,90,41)" rx="2" ry="2" />
<text  x="1153.06" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).TransportProtocolOption (157 samples, 0.05%)</title><rect x="1039.8" y="773" width="0.6" height="15.0" fill="rgb(245,67,48)" rx="2" ry="2" />
<text  x="1042.82" y="783.5" ></text>
</g>
<g >
<title>runtime.morestack (913 samples, 0.28%)</title><rect x="1101.8" y="773" width="3.3" height="15.0" fill="rgb(227,37,28)" rx="2" ry="2" />
<text  x="1104.81" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Get (41 samples, 0.01%)</title><rect x="70.2" y="741" width="0.2" height="15.0" fill="rgb(218,36,25)" rx="2" ry="2" />
<text  x="73.25" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (256 samples, 0.08%)</title><rect x="1188.8" y="629" width="0.9" height="15.0" fill="rgb(207,200,36)" rx="2" ry="2" />
<text  x="1191.78" y="639.5" ></text>
</g>
<g >
<title>runtime.stopm (453 samples, 0.14%)</title><rect x="1186.0" y="805" width="1.6" height="15.0" fill="rgb(229,58,43)" rx="2" ry="2" />
<text  x="1189.00" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).ptraceSyscallEnter (58 samples, 0.02%)</title><rect x="221.0" y="821" width="0.2" height="15.0" fill="rgb(241,90,18)" rx="2" ry="2" />
<text  x="223.97" y="831.5" ></text>
</g>
<g >
<title>runtime.selectnbrecv (53 samples, 0.02%)</title><rect x="162.9" y="725" width="0.2" height="15.0" fill="rgb(245,127,48)" rx="2" ry="2" />
<text  x="165.92" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (196 samples, 0.06%)</title><rect x="1127.9" y="677" width="0.7" height="15.0" fill="rgb(206,173,35)" rx="2" ry="2" />
<text  x="1130.85" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="824.4" y="709" width="0.1" height="15.0" fill="rgb(208,216,30)" rx="2" ry="2" />
<text  x="827.38" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (69 samples, 0.02%)</title><rect x="170.6" y="725" width="0.2" height="15.0" fill="rgb(212,134,28)" rx="2" ry="2" />
<text  x="173.59" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).maybeSendSegment (2,186 samples, 0.67%)</title><rect x="208.0" y="677" width="7.9" height="15.0" fill="rgb(245,174,43)" rx="2" ry="2" />
<text  x="211.00" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (194 samples, 0.06%)</title><rect x="1049.0" y="629" width="0.7" height="15.0" fill="rgb(251,167,7)" rx="2" ry="2" />
<text  x="1052.04" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (88 samples, 0.03%)</title><rect x="149.0" y="565" width="0.4" height="15.0" fill="rgb(254,142,52)" rx="2" ry="2" />
<text  x="152.03" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).Walk (641 samples, 0.20%)</title><rect x="185.4" y="709" width="2.3" height="15.0" fill="rgb(232,164,22)" rx="2" ry="2" />
<text  x="188.42" y="719.5" ></text>
</g>
<g >
<title>runtime.(*mcache).prepareForSweep (65 samples, 0.02%)</title><rect x="128.0" y="597" width="0.2" height="15.0" fill="rgb(248,198,24)" rx="2" ry="2" />
<text  x="130.97" y="607.5" ></text>
</g>
<g >
<title>runtime.systemstack (732 samples, 0.22%)</title><rect x="210.5" y="389" width="2.7" height="15.0" fill="rgb(210,1,32)" rx="2" ry="2" />
<text  x="213.52" y="399.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (31 samples, 0.01%)</title><rect x="904.8" y="565" width="0.1" height="15.0" fill="rgb(212,177,9)" rx="2" ry="2" />
<text  x="907.79" y="575.5" ></text>
</g>
<g >
<title>runtime.findrunnable (426 samples, 0.13%)</title><rect x="1107.6" y="789" width="1.5" height="15.0" fill="rgb(212,213,2)" rx="2" ry="2" />
<text  x="1110.55" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (452 samples, 0.14%)</title><rect x="909.1" y="581" width="1.7" height="15.0" fill="rgb(225,71,5)" rx="2" ry="2" />
<text  x="912.14" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).setAll (58 samples, 0.02%)</title><rect x="46.4" y="693" width="0.2" height="15.0" fill="rgb(236,28,49)" rx="2" ry="2" />
<text  x="49.42" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53,149 samples, 16.29%)</title><rect x="254.4" y="725" width="192.3" height="15.0" fill="rgb(254,133,51)" rx="2" ry="2" />
<text  x="257.45" y="735.5" >[[kernel.kallsyms]]</text>
</g>
<g >
<title>runtime.stopm (128 samples, 0.04%)</title><rect x="157.4" y="629" width="0.4" height="15.0" fill="rgb(249,111,38)" rx="2" ry="2" />
<text  x="160.36" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (51 samples, 0.02%)</title><rect x="1005.9" y="709" width="0.2" height="15.0" fill="rgb(208,209,9)" rx="2" ry="2" />
<text  x="1008.90" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).protocolMainLoop (139 samples, 0.04%)</title><rect x="1044.7" y="869" width="0.5" height="15.0" fill="rgb(218,40,27)" rx="2" ry="2" />
<text  x="1047.72" y="879.5" ></text>
</g>
<g >
<title>runtime.wakep (1,799 samples, 0.55%)</title><rect x="149.9" y="629" width="6.5" height="15.0" fill="rgb(230,79,49)" rx="2" ry="2" />
<text  x="152.86" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header/parse.TCP (123 samples, 0.04%)</title><rect x="793.6" y="725" width="0.4" height="15.0" fill="rgb(222,209,12)" rx="2" ry="2" />
<text  x="796.55" y="735.5" ></text>
</g>
<g >
<title>runtime.goschedImpl (61 samples, 0.02%)</title><rect x="54.4" y="677" width="0.3" height="15.0" fill="rgb(253,48,2)" rx="2" ry="2" />
<text  x="57.44" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/packetsocket.(*endpoint).Capabilities (45 samples, 0.01%)</title><rect x="723.5" y="741" width="0.2" height="15.0" fill="rgb(218,37,13)" rx="2" ry="2" />
<text  x="726.53" y="751.5" ></text>
</g>
<g >
<title>memeqbody (99 samples, 0.03%)</title><rect x="788.5" y="677" width="0.3" height="15.0" fill="rgb(211,218,23)" rx="2" ry="2" />
<text  x="791.49" y="687.5" ></text>
</g>
<g >
<title>runtime.ready (115 samples, 0.04%)</title><rect x="1003.6" y="693" width="0.4" height="15.0" fill="rgb(208,186,52)" rx="2" ry="2" />
<text  x="1006.58" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).Release (598 samples, 0.18%)</title><rect x="64.8" y="709" width="2.1" height="15.0" fill="rgb(223,90,21)" rx="2" ry="2" />
<text  x="67.76" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (899 samples, 0.28%)</title><rect x="124.1" y="533" width="3.3" height="15.0" fill="rgb(234,55,13)" rx="2" ry="2" />
<text  x="127.12" y="543.5" ></text>
</g>
<g >
<title>runtime.epollwait (175 samples, 0.05%)</title><rect x="904.3" y="741" width="0.6" height="15.0" fill="rgb(233,220,30)" rx="2" ry="2" />
<text  x="907.27" y="751.5" ></text>
</g>
<g >
<title>runtime.newobject (812 samples, 0.25%)</title><rect x="846.3" y="821" width="3.0" height="15.0" fill="rgb(244,49,52)" rx="2" ry="2" />
<text  x="849.34" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).IsLoopback (49 samples, 0.02%)</title><rect x="778.9" y="725" width="0.2" height="15.0" fill="rgb(224,12,28)" rx="2" ry="2" />
<text  x="781.92" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="1028.6" y="341" width="0.2" height="15.0" fill="rgb(218,105,36)" rx="2" ry="2" />
<text  x="1031.62" y="351.5" ></text>
</g>
<g >
<title>runtime.park_m (44 samples, 0.01%)</title><rect x="1004.8" y="789" width="0.2" height="15.0" fill="rgb(230,58,21)" rx="2" ry="2" />
<text  x="1007.83" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (32 samples, 0.01%)</title><rect x="192.1" y="613" width="0.1" height="15.0" fill="rgb(220,207,16)" rx="2" ry="2" />
<text  x="195.13" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).SwapAnd (162 samples, 0.05%)</title><rect x="1085.6" y="693" width="0.6" height="15.0" fill="rgb(240,127,36)" rx="2" ry="2" />
<text  x="1088.64" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/fsutil.(*CachingInodeOperations).UnstableAttr (64 samples, 0.02%)</title><rect x="188.4" y="581" width="0.2" height="15.0" fill="rgb(233,37,38)" rx="2" ry="2" />
<text  x="191.39" y="591.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (42 samples, 0.01%)</title><rect x="1036.5" y="629" width="0.2" height="15.0" fill="rgb(219,95,35)" rx="2" ry="2" />
<text  x="1039.51" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (33,352 samples, 10.22%)</title><rect x="517.4" y="773" width="120.6" height="15.0" fill="rgb(249,181,18)" rx="2" ry="2" />
<text  x="520.40" y="783.5" >[[kernel.kallsy..</text>
</g>
<g >
<title>runtime.newobject (43 samples, 0.01%)</title><rect x="202.6" y="677" width="0.2" height="15.0" fill="rgb(246,21,51)" rx="2" ry="2" />
<text  x="205.60" y="687.5" ></text>
</g>
<g >
<title>time.NewTimer (678 samples, 0.21%)</title><rect x="1081.0" y="645" width="2.5" height="15.0" fill="rgb(206,123,54)" rx="2" ry="2" />
<text  x="1084.02" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).postXmit (117 samples, 0.04%)</title><rect x="215.9" y="677" width="0.4" height="15.0" fill="rgb(234,117,32)" rx="2" ry="2" />
<text  x="218.90" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (74 samples, 0.02%)</title><rect x="149.1" y="533" width="0.3" height="15.0" fill="rgb(243,25,6)" rx="2" ry="2" />
<text  x="152.08" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleListenSegment (6,261 samples, 1.92%)</title><rect x="1021.9" y="853" width="22.7" height="15.0" fill="rgb(242,211,50)" rx="2" ry="2" />
<text  x="1024.93" y="863.5" >g..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (31 samples, 0.01%)</title><rect x="725.8" y="645" width="0.1" height="15.0" fill="rgb(221,66,12)" rx="2" ry="2" />
<text  x="728.77" y="655.5" ></text>
</g>
<g >
<title>time.resetTimer (30 samples, 0.01%)</title><rect x="216.1" y="629" width="0.1" height="15.0" fill="rgb(240,216,45)" rx="2" ry="2" />
<text  x="219.12" y="639.5" ></text>
</g>
<g >
<title>runtime.newobject (72 samples, 0.02%)</title><rect x="1083.5" y="661" width="0.2" height="15.0" fill="rgb(225,199,7)" rx="2" ry="2" />
<text  x="1086.49" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (232 samples, 0.07%)</title><rect x="1186.8" y="725" width="0.8" height="15.0" fill="rgb(225,144,10)" rx="2" ry="2" />
<text  x="1189.78" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="1049.6" y="437" width="0.1" height="15.0" fill="rgb(229,137,33)" rx="2" ry="2" />
<text  x="1052.62" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withVecInternalMappings (47 samples, 0.01%)</title><rect x="21.6" y="725" width="0.1" height="15.0" fill="rgb(226,189,31)" rx="2" ry="2" />
<text  x="24.57" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (380 samples, 0.12%)</title><rect x="1063.4" y="661" width="1.4" height="15.0" fill="rgb(227,32,35)" rx="2" ry="2" />
<text  x="1066.42" y="671.5" ></text>
</g>
<g >
<title>runtime.notewakeup (1,717 samples, 0.53%)</title><rect x="150.1" y="597" width="6.2" height="15.0" fill="rgb(219,163,19)" rx="2" ry="2" />
<text  x="153.11" y="607.5" ></text>
</g>
<g >
<title>runtime.wakep (88 samples, 0.03%)</title><rect x="1005.8" y="805" width="0.3" height="15.0" fill="rgb(242,169,16)" rx="2" ry="2" />
<text  x="1008.77" y="815.5" ></text>
</g>
<g >
<title>runtime.selectgo (38 samples, 0.01%)</title><rect x="162.8" y="725" width="0.1" height="15.0" fill="rgb(226,56,25)" rx="2" ry="2" />
<text  x="165.78" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (56 samples, 0.02%)</title><rect x="726.2" y="661" width="0.2" height="15.0" fill="rgb(236,170,14)" rx="2" ry="2" />
<text  x="729.18" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).existingPMAsLocked (28 samples, 0.01%)</title><rect x="184.3" y="677" width="0.1" height="15.0" fill="rgb(229,221,40)" rx="2" ry="2" />
<text  x="187.27" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).destroy-fm (91 samples, 0.03%)</title><rect x="64.0" y="613" width="0.3" height="15.0" fill="rgb(209,23,44)" rx="2" ry="2" />
<text  x="67.02" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (51 samples, 0.02%)</title><rect x="783.3" y="677" width="0.1" height="15.0" fill="rgb(243,107,8)" rx="2" ry="2" />
<text  x="786.25" y="687.5" ></text>
</g>
<g >
<title>sync.(*Pool).Get (37 samples, 0.01%)</title><rect x="1031.4" y="757" width="0.2" height="15.0" fill="rgb(238,14,14)" rx="2" ry="2" />
<text  x="1034.45" y="767.5" ></text>
</g>
<g >
<title>runtime.mapdelete (110 samples, 0.03%)</title><rect x="1020.5" y="821" width="0.4" height="15.0" fill="rgb(248,149,31)" rx="2" ry="2" />
<text  x="1023.53" y="831.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc1 (57 samples, 0.02%)</title><rect x="1020.2" y="773" width="0.2" height="15.0" fill="rgb(216,32,8)" rx="2" ry="2" />
<text  x="1023.16" y="783.5" ></text>
</g>
<g >
<title>runtime.(*lfstack).pop (191 samples, 0.06%)</title><rect x="1138.1" y="709" width="0.7" height="15.0" fill="rgb(230,205,45)" rx="2" ry="2" />
<text  x="1141.13" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).enqueuePacketBuffer (683 samples, 0.21%)</title><rect x="1027.6" y="677" width="2.4" height="15.0" fill="rgb(237,125,7)" rx="2" ry="2" />
<text  x="1030.58" y="687.5" ></text>
</g>
<g >
<title>runtime.binarySearchTree (33 samples, 0.01%)</title><rect x="1133.8" y="757" width="0.1" height="15.0" fill="rgb(221,55,41)" rx="2" ry="2" />
<text  x="1136.83" y="767.5" ></text>
</g>
<g >
<title>runtime.findrunnable (184 samples, 0.06%)</title><rect x="17.5" y="805" width="0.7" height="15.0" fill="rgb(249,139,48)" rx="2" ry="2" />
<text  x="20.53" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).DeliverNetworkPacket (86 samples, 0.03%)</title><rect x="894.9" y="837" width="0.3" height="15.0" fill="rgb(251,116,2)" rx="2" ry="2" />
<text  x="897.89" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.NewDirent (180 samples, 0.06%)</title><rect x="49.6" y="693" width="0.7" height="15.0" fill="rgb(240,217,1)" rx="2" ry="2" />
<text  x="52.60" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).maxReceiveBufferSize (36 samples, 0.01%)</title><rect x="1024.4" y="741" width="0.2" height="15.0" fill="rgb(220,172,27)" rx="2" ry="2" />
<text  x="1027.45" y="751.5" ></text>
</g>
<g >
<title>runtime.walltime1 (28 samples, 0.01%)</title><rect x="16.2" y="821" width="0.1" height="15.0" fill="rgb(222,186,34)" rx="2" ry="2" />
<text  x="19.23" y="831.5" ></text>
</g>
<g >
<title>runtime.newproc (61 samples, 0.02%)</title><rect x="904.0" y="677" width="0.2" height="15.0" fill="rgb(235,25,48)" rx="2" ry="2" />
<text  x="906.99" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (183 samples, 0.06%)</title><rect x="1082.8" y="517" width="0.7" height="15.0" fill="rgb(245,150,53)" rx="2" ry="2" />
<text  x="1085.80" y="527.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (61 samples, 0.02%)</title><rect x="23.1" y="645" width="0.2" height="15.0" fill="rgb(246,136,35)" rx="2" ry="2" />
<text  x="26.05" y="655.5" ></text>
</g>
<g >
<title>runtime.newobject (84 samples, 0.03%)</title><rect x="1068.2" y="821" width="0.3" height="15.0" fill="rgb(248,38,54)" rx="2" ry="2" />
<text  x="1071.16" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/ports.(*PortManager).ReleasePort (156 samples, 0.05%)</title><rect x="1100.1" y="821" width="0.6" height="15.0" fill="rgb(254,201,9)" rx="2" ry="2" />
<text  x="1103.09" y="831.5" ></text>
</g>
<g >
<title>ext4_block_write_begin (40 samples, 0.01%)</title><rect x="199.2" y="293" width="0.2" height="15.0" fill="rgb(248,0,43)" rx="2" ry="2" />
<text  x="202.23" y="303.5" ></text>
</g>
<g >
<title>runtime.funcspdelta (284 samples, 0.09%)</title><rect x="1016.8" y="725" width="1.1" height="15.0" fill="rgb(253,9,4)" rx="2" ry="2" />
<text  x="1019.83" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (316 samples, 0.10%)</title><rect x="14.7" y="709" width="1.2" height="15.0" fill="rgb(234,73,31)" rx="2" ry="2" />
<text  x="17.72" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (222 samples, 0.07%)</title><rect x="115.1" y="533" width="0.8" height="15.0" fill="rgb(237,13,21)" rx="2" ry="2" />
<text  x="118.08" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/packetsocket.(*endpoint).DeliverNetworkPacket (79 samples, 0.02%)</title><rect x="717.8" y="821" width="0.3" height="15.0" fill="rgb(211,65,27)" rx="2" ry="2" />
<text  x="720.83" y="831.5" ></text>
</g>
<g >
<title>runtime.gopark (135 samples, 0.04%)</title><rect x="86.1" y="709" width="0.5" height="15.0" fill="rgb(225,51,43)" rx="2" ry="2" />
<text  x="89.07" y="719.5" ></text>
</g>
<g >
<title>runtime.systemstack (100 samples, 0.03%)</title><rect x="848.1" y="725" width="0.4" height="15.0" fill="rgb(227,178,46)" rx="2" ry="2" />
<text  x="851.09" y="735.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (727 samples, 0.22%)</title><rect x="210.5" y="373" width="2.7" height="15.0" fill="rgb(250,30,6)" rx="2" ry="2" />
<text  x="213.53" y="383.5" ></text>
</g>
<g >
<title>runtime.convTslice (38 samples, 0.01%)</title><rect x="764.1" y="613" width="0.2" height="15.0" fill="rgb(248,207,20)" rx="2" ry="2" />
<text  x="767.15" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (81 samples, 0.02%)</title><rect x="23.0" y="693" width="0.3" height="15.0" fill="rgb(242,157,30)" rx="2" ry="2" />
<text  x="25.98" y="703.5" ></text>
</g>
<g >
<title>runtime.clone (47 samples, 0.01%)</title><rect x="16.8" y="773" width="0.1" height="15.0" fill="rgb(244,114,17)" rx="2" ry="2" />
<text  x="19.75" y="783.5" ></text>
</g>
<g >
<title>runtime.newobject (91 samples, 0.03%)</title><rect x="1068.5" y="837" width="0.3" height="15.0" fill="rgb(230,141,52)" rx="2" ry="2" />
<text  x="1071.51" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).get (49 samples, 0.02%)</title><rect x="181.6" y="469" width="0.2" height="15.0" fill="rgb(216,189,5)" rx="2" ry="2" />
<text  x="184.64" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleWrite (878 samples, 0.27%)</title><rect x="179.9" y="693" width="3.1" height="15.0" fill="rgb(248,117,16)" rx="2" ry="2" />
<text  x="182.85" y="703.5" ></text>
</g>
<g >
<title>runtime.clone (878 samples, 0.27%)</title><rect x="12.7" y="821" width="3.2" height="15.0" fill="rgb(246,68,8)" rx="2" ry="2" />
<text  x="15.72" y="831.5" ></text>
</g>
<g >
<title>runtime.unlock2 (71 samples, 0.02%)</title><rect x="1153.3" y="709" width="0.2" height="15.0" fill="rgb(250,220,10)" rx="2" ry="2" />
<text  x="1156.26" y="719.5" ></text>
</g>
<g >
<title>memeqbody (113 samples, 0.03%)</title><rect x="772.2" y="629" width="0.4" height="15.0" fill="rgb(212,168,32)" rx="2" ry="2" />
<text  x="775.19" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (41 samples, 0.01%)</title><rect x="60.9" y="741" width="0.2" height="15.0" fill="rgb(245,107,37)" rx="2" ry="2" />
<text  x="63.91" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (174 samples, 0.05%)</title><rect x="1187.0" y="661" width="0.6" height="15.0" fill="rgb(235,199,20)" rx="2" ry="2" />
<text  x="1189.99" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (275 samples, 0.08%)</title><rect x="703.9" y="693" width="1.0" height="15.0" fill="rgb(252,141,3)" rx="2" ry="2" />
<text  x="706.88" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).SwapAnd (224 samples, 0.07%)</title><rect x="161.9" y="725" width="0.9" height="15.0" fill="rgb(212,27,31)" rx="2" ry="2" />
<text  x="164.94" y="735.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (44 samples, 0.01%)</title><rect x="207.4" y="629" width="0.2" height="15.0" fill="rgb(233,217,4)" rx="2" ry="2" />
<text  x="210.41" y="639.5" ></text>
</g>
<g >
<title>runtime.notesleep (126 samples, 0.04%)</title><rect x="1007.2" y="725" width="0.5" height="15.0" fill="rgb(207,188,47)" rx="2" ry="2" />
<text  x="1010.20" y="735.5" ></text>
</g>
<g >
<title>runtime.schedule (6,830 samples, 2.09%)</title><rect x="897.6" y="773" width="24.7" height="15.0" fill="rgb(231,150,41)" rx="2" ry="2" />
<text  x="900.61" y="783.5" >r..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (122 samples, 0.04%)</title><rect x="702.4" y="677" width="0.5" height="15.0" fill="rgb(221,117,49)" rx="2" ry="2" />
<text  x="705.41" y="687.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (74 samples, 0.02%)</title><rect x="452.2" y="773" width="0.3" height="15.0" fill="rgb(232,178,23)" rx="2" ry="2" />
<text  x="455.24" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (29 samples, 0.01%)</title><rect x="1154.4" y="693" width="0.1" height="15.0" fill="rgb(240,162,44)" rx="2" ry="2" />
<text  x="1157.44" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*endpoint).AddHeader (60 samples, 0.02%)</title><rect x="926.1" y="805" width="0.2" height="15.0" fill="rgb(251,61,45)" rx="2" ry="2" />
<text  x="929.09" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (46 samples, 0.01%)</title><rect x="996.5" y="405" width="0.1" height="15.0" fill="rgb(243,162,11)" rx="2" ry="2" />
<text  x="999.47" y="415.5" ></text>
</g>
<g >
<title>runtime.mallocgc (30 samples, 0.01%)</title><rect x="204.4" y="709" width="0.1" height="15.0" fill="rgb(233,111,9)" rx="2" ry="2" />
<text  x="207.39" y="719.5" ></text>
</g>
<g >
<title>runtime.park_m (1,356 samples, 0.42%)</title><rect x="1185.1" y="853" width="4.9" height="15.0" fill="rgb(236,96,25)" rx="2" ry="2" />
<text  x="1188.10" y="863.5" ></text>
</g>
<g >
<title>runtime.park_m (6,905 samples, 2.12%)</title><rect x="897.3" y="789" width="25.0" height="15.0" fill="rgb(211,155,49)" rx="2" ry="2" />
<text  x="900.34" y="799.5" >r..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (65 samples, 0.02%)</title><rect x="149.1" y="517" width="0.3" height="15.0" fill="rgb(212,136,0)" rx="2" ry="2" />
<text  x="152.11" y="527.5" ></text>
</g>
<g >
<title>runtime.mallocgc (91 samples, 0.03%)</title><rect x="1094.1" y="741" width="0.4" height="15.0" fill="rgb(232,24,38)" rx="2" ry="2" />
<text  x="1097.15" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (48 samples, 0.01%)</title><rect x="13.2" y="549" width="0.2" height="15.0" fill="rgb(214,56,47)" rx="2" ry="2" />
<text  x="16.22" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IncRef (113 samples, 0.03%)</title><rect x="726.2" y="677" width="0.4" height="15.0" fill="rgb(205,94,40)" rx="2" ry="2" />
<text  x="729.16" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (311 samples, 0.10%)</title><rect x="14.7" y="693" width="1.2" height="15.0" fill="rgb(229,16,51)" rx="2" ry="2" />
<text  x="17.74" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Get (220 samples, 0.07%)</title><rect x="43.9" y="741" width="0.8" height="15.0" fill="rgb(226,48,9)" rx="2" ry="2" />
<text  x="46.95" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,983 samples, 0.61%)</title><rect x="194.2" y="469" width="7.2" height="15.0" fill="rgb(211,18,39)" rx="2" ry="2" />
<text  x="197.23" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendTCP (1,974 samples, 0.60%)</title><rect x="208.4" y="613" width="7.2" height="15.0" fill="rgb(209,104,6)" rx="2" ry="2" />
<text  x="211.45" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (76 samples, 0.02%)</title><rect x="19.7" y="581" width="0.2" height="15.0" fill="rgb(242,152,28)" rx="2" ry="2" />
<text  x="22.66" y="591.5" ></text>
</g>
<g >
<title>runtime.(*pageAlloc).scavenge (243 samples, 0.07%)</title><rect x="1130.1" y="837" width="0.9" height="15.0" fill="rgb(227,158,8)" rx="2" ry="2" />
<text  x="1133.08" y="847.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (93 samples, 0.03%)</title><rect x="65.6" y="597" width="0.4" height="15.0" fill="rgb(218,178,32)" rx="2" ry="2" />
<text  x="68.64" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/buffer.(*VectorisedView).CapLength (38 samples, 0.01%)</title><rect x="780.7" y="709" width="0.1" height="15.0" fill="rgb(228,19,37)" rx="2" ry="2" />
<text  x="783.68" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/limits.(*LimitSet).Get (47 samples, 0.01%)</title><rect x="46.7" y="693" width="0.2" height="15.0" fill="rgb(240,183,52)" rx="2" ry="2" />
<text  x="49.70" y="703.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (99 samples, 0.03%)</title><rect x="729.0" y="677" width="0.3" height="15.0" fill="rgb(213,69,12)" rx="2" ry="2" />
<text  x="731.95" y="687.5" ></text>
</g>
<g >
<title>runtime.newproc (78 samples, 0.02%)</title><rect x="114.1" y="565" width="0.3" height="15.0" fill="rgb(243,150,38)" rx="2" ry="2" />
<text  x="117.13" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (405 samples, 0.12%)</title><rect x="705.2" y="661" width="1.5" height="15.0" fill="rgb(230,83,11)" rx="2" ry="2" />
<text  x="708.23" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.makeRoute (310 samples, 0.10%)</title><rect x="1033.2" y="741" width="1.2" height="15.0" fill="rgb(249,191,13)" rx="2" ry="2" />
<text  x="1036.23" y="751.5" ></text>
</g>
<g >
<title>runtime.getempty (54 samples, 0.02%)</title><rect x="1138.8" y="709" width="0.2" height="15.0" fill="rgb(210,29,22)" rx="2" ry="2" />
<text  x="1141.84" y="719.5" ></text>
</g>
<g >
<title>runtime.(*pallocBits).summarize (29 samples, 0.01%)</title><rect x="1098.4" y="645" width="0.1" height="15.0" fill="rgb(239,226,18)" rx="2" ry="2" />
<text  x="1101.43" y="655.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (37 samples, 0.01%)</title><rect x="773.3" y="661" width="0.1" height="15.0" fill="rgb(237,132,37)" rx="2" ry="2" />
<text  x="776.27" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.BlockSeq.DropFirst64 (38 samples, 0.01%)</title><rect x="76.3" y="677" width="0.1" height="15.0" fill="rgb(244,15,50)" rx="2" ry="2" />
<text  x="79.28" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).WritePacket (189 samples, 0.06%)</title><rect x="1073.7" y="597" width="0.7" height="15.0" fill="rgb(244,15,37)" rx="2" ry="2" />
<text  x="1076.73" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (52 samples, 0.02%)</title><rect x="702.7" y="581" width="0.2" height="15.0" fill="rgb(223,181,27)" rx="2" ry="2" />
<text  x="705.67" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*endpointsByNIC).unregisterEndpoint (97 samples, 0.03%)</title><rect x="1101.2" y="773" width="0.3" height="15.0" fill="rgb(250,205,11)" rx="2" ry="2" />
<text  x="1104.16" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).NowNanoseconds (48 samples, 0.01%)</title><rect x="1078.6" y="725" width="0.2" height="15.0" fill="rgb(213,130,30)" rx="2" ry="2" />
<text  x="1081.62" y="735.5" ></text>
</g>
<g >
<title>runtime.siftupTimer (40 samples, 0.01%)</title><rect x="1082.2" y="581" width="0.2" height="15.0" fill="rgb(232,14,54)" rx="2" ry="2" />
<text  x="1085.25" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (199 samples, 0.06%)</title><rect x="997.4" y="613" width="0.7" height="15.0" fill="rgb(215,55,10)" rx="2" ry="2" />
<text  x="1000.41" y="623.5" ></text>
</g>
<g >
<title>runtime.cleantimers (91 samples, 0.03%)</title><rect x="1081.8" y="597" width="0.4" height="15.0" fill="rgb(210,212,52)" rx="2" ry="2" />
<text  x="1084.83" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).getAddressOrCreateTemp (77 samples, 0.02%)</title><rect x="1032.8" y="741" width="0.3" height="15.0" fill="rgb(242,77,31)" rx="2" ry="2" />
<text  x="1035.83" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Readiness (67 samples, 0.02%)</title><rect x="167.1" y="725" width="0.3" height="15.0" fill="rgb(253,95,14)" rx="2" ry="2" />
<text  x="170.14" y="735.5" ></text>
</g>
<g >
<title>syscall.Wait4 (282 samples, 0.09%)</title><rect x="703.9" y="789" width="1.0" height="15.0" fill="rgb(240,7,44)" rx="2" ry="2" />
<text  x="706.86" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (49 samples, 0.02%)</title><rect x="195.2" y="405" width="0.2" height="15.0" fill="rgb(216,145,52)" rx="2" ry="2" />
<text  x="198.25" y="415.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/rawfile.NonBlockingSendMMsg (49 samples, 0.02%)</title><rect x="998.1" y="821" width="0.2" height="15.0" fill="rgb(246,163,10)" rx="2" ry="2" />
<text  x="1001.14" y="831.5" ></text>
</g>
<g >
<title>runtime.schedule (88 samples, 0.03%)</title><rect x="17.2" y="821" width="0.3" height="15.0" fill="rgb(206,126,50)" rx="2" ry="2" />
<text  x="20.18" y="831.5" ></text>
</g>
<g >
<title>runtime.mapassign_fast64ptr (39 samples, 0.01%)</title><rect x="49.7" y="661" width="0.1" height="15.0" fill="rgb(213,87,24)" rx="2" ry="2" />
<text  x="52.66" y="671.5" ></text>
</g>
<g >
<title>runtime.(*mheap).freeSpanLocked (123 samples, 0.04%)</title><rect x="1110.5" y="789" width="0.4" height="15.0" fill="rgb(250,132,5)" rx="2" ry="2" />
<text  x="1113.50" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="18.0" y="693" width="0.2" height="15.0" fill="rgb(250,150,19)" rx="2" ry="2" />
<text  x="21.03" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (71 samples, 0.02%)</title><rect x="1083.2" y="389" width="0.3" height="15.0" fill="rgb(205,50,36)" rx="2" ry="2" />
<text  x="1086.21" y="399.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportEndpoints).findEndpointLocked (95 samples, 0.03%)</title><rect x="1067.5" y="805" width="0.3" height="15.0" fill="rgb(229,127,8)" rx="2" ry="2" />
<text  x="1070.46" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (94 samples, 0.03%)</title><rect x="766.1" y="501" width="0.4" height="15.0" fill="rgb(229,194,54)" rx="2" ry="2" />
<text  x="769.15" y="511.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (179 samples, 0.05%)</title><rect x="700.9" y="645" width="0.7" height="15.0" fill="rgb(223,138,23)" rx="2" ry="2" />
<text  x="703.94" y="655.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast32 (29 samples, 0.01%)</title><rect x="456.2" y="805" width="0.1" height="15.0" fill="rgb(242,75,49)" rx="2" ry="2" />
<text  x="459.23" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/binary.marshal (98 samples, 0.03%)</title><rect x="924.6" y="773" width="0.4" height="15.0" fill="rgb(216,5,22)" rx="2" ry="2" />
<text  x="927.63" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (58 samples, 0.02%)</title><rect x="1062.3" y="581" width="0.2" height="15.0" fill="rgb(232,132,40)" rx="2" ry="2" />
<text  x="1065.29" y="591.5" ></text>
</g>
<g >
<title>runtime.lock2 (46 samples, 0.01%)</title><rect x="1119.7" y="805" width="0.2" height="15.0" fill="rgb(253,172,3)" rx="2" ry="2" />
<text  x="1122.73" y="815.5" ></text>
</g>
<g >
<title>runtime.newobject (52 samples, 0.02%)</title><rect x="215.4" y="581" width="0.2" height="15.0" fill="rgb(217,49,35)" rx="2" ry="2" />
<text  x="218.38" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendSegmentFromView (2,709 samples, 0.83%)</title><rect x="1089.6" y="821" width="9.8" height="15.0" fill="rgb(212,90,32)" rx="2" ry="2" />
<text  x="1092.64" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (198 samples, 0.06%)</title><rect x="1119.0" y="773" width="0.7" height="15.0" fill="rgb(235,180,45)" rx="2" ry="2" />
<text  x="1122.01" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (63 samples, 0.02%)</title><rect x="702.6" y="597" width="0.3" height="15.0" fill="rgb(208,172,52)" rx="2" ry="2" />
<text  x="705.63" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (3,558 samples, 1.09%)</title><rect x="745.6" y="613" width="12.9" height="15.0" fill="rgb(210,91,36)" rx="2" ry="2" />
<text  x="748.62" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).accountTaskGoroutineLeave (407 samples, 0.12%)</title><rect x="81.4" y="693" width="1.4" height="15.0" fill="rgb(206,126,7)" rx="2" ry="2" />
<text  x="84.37" y="703.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (513 samples, 0.16%)</title><rect x="765.3" y="549" width="1.8" height="15.0" fill="rgb(221,52,8)" rx="2" ry="2" />
<text  x="768.26" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).setEndpointState (35 samples, 0.01%)</title><rect x="180.0" y="645" width="0.2" height="15.0" fill="rgb(222,125,23)" rx="2" ry="2" />
<text  x="183.05" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (116 samples, 0.04%)</title><rect x="695.6" y="805" width="0.4" height="15.0" fill="rgb(246,103,4)" rx="2" ry="2" />
<text  x="698.61" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (77 samples, 0.02%)</title><rect x="757.8" y="309" width="0.3" height="15.0" fill="rgb(243,132,7)" rx="2" ry="2" />
<text  x="760.79" y="319.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (45 samples, 0.01%)</title><rect x="701.8" y="629" width="0.2" height="15.0" fill="rgb(248,36,35)" rx="2" ry="2" />
<text  x="704.84" y="639.5" ></text>
</g>
<g >
<title>runtime.newobject (242 samples, 0.07%)</title><rect x="929.5" y="805" width="0.9" height="15.0" fill="rgb(216,156,3)" rx="2" ry="2" />
<text  x="932.54" y="815.5" ></text>
</g>
<g >
<title>runtime.futex (123 samples, 0.04%)</title><rect x="1007.2" y="709" width="0.5" height="15.0" fill="rgb(231,37,51)" rx="2" ry="2" />
<text  x="1010.21" y="719.5" ></text>
</g>
<g >
<title>runtime.startm (364 samples, 0.11%)</title><rect x="1127.3" y="789" width="1.3" height="15.0" fill="rgb(254,122,23)" rx="2" ry="2" />
<text  x="1130.25" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="1108.8" y="581" width="0.2" height="15.0" fill="rgb(236,223,24)" rx="2" ry="2" />
<text  x="1111.84" y="591.5" ></text>
</g>
<g >
<title>runtime.reentersyscall (42 samples, 0.01%)</title><rect x="450.6" y="757" width="0.2" height="15.0" fill="rgb(218,29,43)" rx="2" ry="2" />
<text  x="453.64" y="767.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (12,284 samples, 3.76%)</title><rect x="850.5" y="821" width="44.4" height="15.0" fill="rgb(230,8,43)" rx="2" ry="2" />
<text  x="853.47" y="831.5" >sysc..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (29 samples, 0.01%)</title><rect x="725.2" y="677" width="0.1" height="15.0" fill="rgb(225,22,29)" rx="2" ry="2" />
<text  x="728.22" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/iovec.(*Builder).Add (50 samples, 0.02%)</title><rect x="925.9" y="805" width="0.2" height="15.0" fill="rgb(230,82,32)" rx="2" ry="2" />
<text  x="928.91" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (29 samples, 0.01%)</title><rect x="1185.9" y="725" width="0.1" height="15.0" fill="rgb(211,215,33)" rx="2" ry="2" />
<text  x="1188.89" y="735.5" ></text>
</g>
<g >
<title>runtime.schedule (4,337 samples, 1.33%)</title><rect x="1113.2" y="837" width="15.7" height="15.0" fill="rgb(209,15,15)" rx="2" ry="2" />
<text  x="1116.25" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (265 samples, 0.08%)</title><rect x="920.9" y="581" width="1.0" height="15.0" fill="rgb(217,130,33)" rx="2" ry="2" />
<text  x="923.92" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (183 samples, 0.06%)</title><rect x="990.1" y="101" width="0.7" height="15.0" fill="rgb(218,146,40)" rx="2" ry="2" />
<text  x="993.13" y="111.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).accountTaskGoroutineEnter (133 samples, 0.04%)</title><rect x="83.5" y="693" width="0.5" height="15.0" fill="rgb(216,70,53)" rx="2" ry="2" />
<text  x="86.48" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (84 samples, 0.03%)</title><rect x="56.8" y="661" width="0.3" height="15.0" fill="rgb(212,174,52)" rx="2" ry="2" />
<text  x="59.79" y="671.5" ></text>
</g>
<g >
<title>runtime.cleantimers (57 samples, 0.02%)</title><rect x="1069.1" y="789" width="0.2" height="15.0" fill="rgb(205,76,30)" rx="2" ry="2" />
<text  x="1072.06" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,974 samples, 1.83%)</title><rect x="873.3" y="709" width="21.6" height="15.0" fill="rgb(218,61,39)" rx="2" ry="2" />
<text  x="876.28" y="719.5" >[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).Value (35 samples, 0.01%)</title><rect x="52.3" y="677" width="0.1" height="15.0" fill="rgb(213,0,12)" rx="2" ry="2" />
<text  x="55.30" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*WeakRef).init (76 samples, 0.02%)</title><rect x="45.4" y="693" width="0.3" height="15.0" fill="rgb(239,4,49)" rx="2" ry="2" />
<text  x="48.38" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacketBuffer (972 samples, 0.30%)</title><rect x="209.9" y="501" width="3.5" height="15.0" fill="rgb(233,126,52)" rx="2" ry="2" />
<text  x="212.90" y="511.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).entry (43 samples, 0.01%)</title><rect x="181.7" y="453" width="0.1" height="15.0" fill="rgb(208,206,24)" rx="2" ry="2" />
<text  x="184.66" y="463.5" ></text>
</g>
<g >
<title>runtime.usleep (36 samples, 0.01%)</title><rect x="29.4" y="869" width="0.1" height="15.0" fill="rgb(231,13,23)" rx="2" ry="2" />
<text  x="32.38" y="879.5" ></text>
</g>
<g >
<title>runtime.mallocgc (39 samples, 0.01%)</title><rect x="1023.0" y="741" width="0.2" height="15.0" fill="rgb(233,38,26)" rx="2" ry="2" />
<text  x="1026.04" y="751.5" ></text>
</g>
<g >
<title>hash_conntrack_raw (60 samples, 0.02%)</title><rect x="961.7" y="453" width="0.2" height="15.0" fill="rgb(231,94,7)" rx="2" ry="2" />
<text  x="964.68" y="463.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBuffer).Network (349 samples, 0.11%)</title><rect x="762.9" y="613" width="1.2" height="15.0" fill="rgb(222,157,0)" rx="2" ry="2" />
<text  x="765.89" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).WritePacket (689 samples, 0.21%)</title><rect x="1027.6" y="693" width="2.5" height="15.0" fill="rgb(207,194,0)" rx="2" ry="2" />
<text  x="1030.57" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1108.9" y="549" width="0.1" height="15.0" fill="rgb(216,213,23)" rx="2" ry="2" />
<text  x="1111.94" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (8,155 samples, 2.50%)</title><rect x="865.4" y="789" width="29.5" height="15.0" fill="rgb(225,129,43)" rx="2" ry="2" />
<text  x="868.40" y="799.5" >[[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).deliverAccepted (1,194 samples, 0.37%)</title><rect x="1000.1" y="853" width="4.4" height="15.0" fill="rgb(215,52,44)" rx="2" ry="2" />
<text  x="1003.14" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (70 samples, 0.02%)</title><rect x="81.0" y="661" width="0.3" height="15.0" fill="rgb(246,189,51)" rx="2" ry="2" />
<text  x="84.03" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (43 samples, 0.01%)</title><rect x="719.5" y="757" width="0.1" height="15.0" fill="rgb(219,35,0)" rx="2" ry="2" />
<text  x="722.48" y="767.5" ></text>
</g>
<g >
<title>runtime.chanrecv (58 samples, 0.02%)</title><rect x="161.4" y="693" width="0.2" height="15.0" fill="rgb(235,157,46)" rx="2" ry="2" />
<text  x="164.39" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (63 samples, 0.02%)</title><rect x="905.3" y="725" width="0.2" height="15.0" fill="rgb(221,97,40)" rx="2" ry="2" />
<text  x="908.30" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (33 samples, 0.01%)</title><rect x="1019.6" y="789" width="0.1" height="15.0" fill="rgb(212,95,28)" rx="2" ry="2" />
<text  x="1022.59" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).WritePacket (254 samples, 0.08%)</title><rect x="1028.0" y="597" width="1.0" height="15.0" fill="rgb(227,104,14)" rx="2" ry="2" />
<text  x="1031.03" y="607.5" ></text>
</g>
<g >
<title>runtime.startm (107 samples, 0.03%)</title><rect x="19.0" y="837" width="0.4" height="15.0" fill="rgb(219,202,6)" rx="2" ry="2" />
<text  x="22.03" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).WritePacket (355 samples, 0.11%)</title><rect x="1027.8" y="629" width="1.2" height="15.0" fill="rgb(209,38,15)" rx="2" ry="2" />
<text  x="1030.75" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).NewFDFrom (226 samples, 0.07%)</title><rect x="46.3" y="725" width="0.8" height="15.0" fill="rgb(208,148,35)" rx="2" ry="2" />
<text  x="49.31" y="735.5" ></text>
</g>
<g >
<title>runtime.systemstack (138 samples, 0.04%)</title><rect x="1110.5" y="821" width="0.5" height="15.0" fill="rgb(206,103,20)" rx="2" ry="2" />
<text  x="1113.49" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,249 samples, 1.30%)</title><rect x="431.3" y="645" width="15.4" height="15.0" fill="rgb(224,156,54)" rx="2" ry="2" />
<text  x="434.29" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (24,558 samples, 7.53%)</title><rect x="549.2" y="725" width="88.8" height="15.0" fill="rgb(230,104,31)" rx="2" ry="2" />
<text  x="552.21" y="735.5" >[[kernel.k..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.overlayUnstableAttr (101 samples, 0.03%)</title><rect x="191.1" y="693" width="0.4" height="15.0" fill="rgb(234,1,30)" rx="2" ry="2" />
<text  x="194.14" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="1120.2" y="757" width="0.2" height="15.0" fill="rgb(247,149,33)" rx="2" ry="2" />
<text  x="1123.19" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls.WaitEpoll (26,484 samples, 8.12%)</title><rect x="77.5" y="757" width="95.8" height="15.0" fill="rgb(222,1,15)" rx="2" ry="2" />
<text  x="80.49" y="767.5" >gvisor.dev/..</text>
</g>
<g >
<title>runtime.memhash32 (69 samples, 0.02%)</title><rect x="453.7" y="789" width="0.3" height="15.0" fill="rgb(218,52,1)" rx="2" ry="2" />
<text  x="456.73" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (531 samples, 0.16%)</title><rect x="1187.8" y="741" width="1.9" height="15.0" fill="rgb(231,72,42)" rx="2" ry="2" />
<text  x="1190.79" y="751.5" ></text>
</g>
<g >
<title>runtime.makechan (50 samples, 0.02%)</title><rect x="66.3" y="693" width="0.2" height="15.0" fill="rgb(231,62,41)" rx="2" ry="2" />
<text  x="69.35" y="703.5" ></text>
</g>
<g >
<title>runtime.stackpoolalloc (231 samples, 0.07%)</title><rect x="1097.9" y="709" width="0.9" height="15.0" fill="rgb(216,158,8)" rx="2" ry="2" />
<text  x="1100.93" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (143 samples, 0.04%)</title><rect x="1049.2" y="581" width="0.5" height="15.0" fill="rgb(219,54,9)" rx="2" ry="2" />
<text  x="1052.23" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (6,424 samples, 1.97%)</title><rect x="969.4" y="309" width="23.2" height="15.0" fill="rgb(240,147,48)" rx="2" ry="2" />
<text  x="972.36" y="319.5" >[..</text>
</g>
<g >
<title>runtime.systemstack (248 samples, 0.08%)</title><rect x="1080.1" y="645" width="0.9" height="15.0" fill="rgb(246,66,7)" rx="2" ry="2" />
<text  x="1083.12" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (365 samples, 0.11%)</title><rect x="114.6" y="613" width="1.3" height="15.0" fill="rgb(229,16,10)" rx="2" ry="2" />
<text  x="117.56" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (144 samples, 0.04%)</title><rect x="451.7" y="485" width="0.5" height="15.0" fill="rgb(252,22,22)" rx="2" ry="2" />
<text  x="454.72" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (140 samples, 0.04%)</title><rect x="65.5" y="629" width="0.5" height="15.0" fill="rgb(247,126,35)" rx="2" ry="2" />
<text  x="68.53" y="639.5" ></text>
</g>
<g >
<title>runtime.(*pageAlloc).alloc (71 samples, 0.02%)</title><rect x="1098.4" y="677" width="0.2" height="15.0" fill="rgb(210,64,9)" rx="2" ry="2" />
<text  x="1101.35" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress (59 samples, 0.02%)</title><rect x="1032.9" y="709" width="0.2" height="15.0" fill="rgb(241,138,38)" rx="2" ry="2" />
<text  x="1035.88" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).makeOptions (39 samples, 0.01%)</title><rect x="1072.1" y="741" width="0.1" height="15.0" fill="rgb(223,174,43)" rx="2" ry="2" />
<text  x="1075.10" y="751.5" ></text>
</g>
<g >
<title>runtime.systemstack (138 samples, 0.04%)</title><rect x="1003.5" y="725" width="0.5" height="15.0" fill="rgb(211,223,25)" rx="2" ry="2" />
<text  x="1006.52" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (46 samples, 0.01%)</title><rect x="52.8" y="677" width="0.1" height="15.0" fill="rgb(243,173,3)" rx="2" ry="2" />
<text  x="55.78" y="687.5" ></text>
</g>
<g >
<title>runtime.schedule (429 samples, 0.13%)</title><rect x="705.2" y="789" width="1.5" height="15.0" fill="rgb(222,123,23)" rx="2" ry="2" />
<text  x="708.15" y="799.5" ></text>
</g>
<g >
<title>runtime.releaseSudog (98 samples, 0.03%)</title><rect x="158.8" y="693" width="0.4" height="15.0" fill="rgb(227,64,39)" rx="2" ry="2" />
<text  x="161.82" y="703.5" ></text>
</g>
<g >
<title>runtime.reentersyscall (411 samples, 0.13%)</title><rect x="446.8" y="741" width="1.5" height="15.0" fill="rgb(230,225,42)" rx="2" ry="2" />
<text  x="449.84" y="751.5" ></text>
</g>
<g >
<title>crc32c_pcl_intel_update (71 samples, 0.02%)</title><rect x="201.4" y="485" width="0.3" height="15.0" fill="rgb(234,52,17)" rx="2" ry="2" />
<text  x="204.40" y="495.5" ></text>
</g>
<g >
<title>runtime.assertI2I2 (138 samples, 0.04%)</title><rect x="58.0" y="741" width="0.5" height="15.0" fill="rgb(231,118,35)" rx="2" ry="2" />
<text  x="61.00" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).IsLoopback (131 samples, 0.04%)</title><rect x="738.6" y="693" width="0.5" height="15.0" fill="rgb(225,125,48)" rx="2" ry="2" />
<text  x="741.59" y="703.5" ></text>
</g>
<g >
<title>runtime.acquirep (108 samples, 0.03%)</title><rect x="128.0" y="613" width="0.3" height="15.0" fill="rgb(207,29,34)" rx="2" ry="2" />
<text  x="130.96" y="623.5" ></text>
</g>
<g >
<title>runtime.wakep (2,924 samples, 0.90%)</title><rect x="747.6" y="549" width="10.6" height="15.0" fill="rgb(238,192,9)" rx="2" ry="2" />
<text  x="750.61" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,034 samples, 0.32%)</title><rect x="152.6" y="501" width="3.7" height="15.0" fill="rgb(216,181,0)" rx="2" ry="2" />
<text  x="155.55" y="511.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (364 samples, 0.11%)</title><rect x="1063.5" y="645" width="1.3" height="15.0" fill="rgb(225,161,47)" rx="2" ry="2" />
<text  x="1066.48" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="452.3" y="581" width="0.2" height="15.0" fill="rgb(228,74,38)" rx="2" ry="2" />
<text  x="455.31" y="591.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (59 samples, 0.02%)</title><rect x="1013.0" y="725" width="0.3" height="15.0" fill="rgb(249,2,42)" rx="2" ry="2" />
<text  x="1016.04" y="735.5" ></text>
</g>
<g >
<title>runtime.suspendG (48 samples, 0.01%)</title><rect x="1154.6" y="789" width="0.2" height="15.0" fill="rgb(220,60,12)" rx="2" ry="2" />
<text  x="1157.58" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).DecRef (61 samples, 0.02%)</title><rect x="78.3" y="741" width="0.2" height="15.0" fill="rgb(222,13,25)" rx="2" ry="2" />
<text  x="81.28" y="751.5" ></text>
</g>
<g >
<title>runtime.makeslice (29 samples, 0.01%)</title><rect x="215.2" y="581" width="0.1" height="15.0" fill="rgb(221,204,29)" rx="2" ry="2" />
<text  x="218.19" y="591.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (30 samples, 0.01%)</title><rect x="184.7" y="677" width="0.1" height="15.0" fill="rgb(221,88,16)" rx="2" ry="2" />
<text  x="187.66" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (30 samples, 0.01%)</title><rect x="1154.4" y="709" width="0.1" height="15.0" fill="rgb(239,5,28)" rx="2" ry="2" />
<text  x="1157.43" y="719.5" ></text>
</g>
<g >
<title>runtime.makeslice (33 samples, 0.01%)</title><rect x="1031.0" y="741" width="0.1" height="15.0" fill="rgb(223,14,40)" rx="2" ry="2" />
<text  x="1034.00" y="751.5" ></text>
</g>
<g >
<title>runtime.systemstack (174 samples, 0.05%)</title><rect x="1028.2" y="549" width="0.6" height="15.0" fill="rgb(252,140,45)" rx="2" ry="2" />
<text  x="1031.17" y="559.5" ></text>
</g>
<g >
<title>runtime.growslice (374 samples, 0.11%)</title><rect x="167.4" y="725" width="1.3" height="15.0" fill="rgb(208,108,28)" rx="2" ry="2" />
<text  x="170.38" y="735.5" ></text>
</g>
<g >
<title>runtime.wakep (66 samples, 0.02%)</title><rect x="65.7" y="565" width="0.3" height="15.0" fill="rgb(225,61,21)" rx="2" ry="2" />
<text  x="68.74" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (165 samples, 0.05%)</title><rect x="1064.2" y="533" width="0.6" height="15.0" fill="rgb(253,174,38)" rx="2" ry="2" />
<text  x="1067.20" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).Writev (52 samples, 0.02%)</title><rect x="21.6" y="805" width="0.2" height="15.0" fill="rgb(206,74,53)" rx="2" ry="2" />
<text  x="24.57" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (393 samples, 0.12%)</title><rect x="211.7" y="229" width="1.4" height="15.0" fill="rgb(247,157,14)" rx="2" ry="2" />
<text  x="214.69" y="239.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="1154.4" y="725" width="0.1" height="15.0" fill="rgb(215,129,2)" rx="2" ry="2" />
<text  x="1157.41" y="735.5" ></text>
</g>
<g >
<title>runtime.memmove (133 samples, 0.04%)</title><rect x="1018.6" y="757" width="0.5" height="15.0" fill="rgb(228,19,3)" rx="2" ry="2" />
<text  x="1021.64" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).TransportProtocolOption (43 samples, 0.01%)</title><rect x="1067.1" y="837" width="0.2" height="15.0" fill="rgb(211,105,16)" rx="2" ry="2" />
<text  x="1070.12" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Watches).Notify (29 samples, 0.01%)</title><rect x="190.4" y="725" width="0.1" height="15.0" fill="rgb(226,41,48)" rx="2" ry="2" />
<text  x="193.40" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*dispatcher).queuePacket (92 samples, 0.03%)</title><rect x="19.7" y="741" width="0.3" height="15.0" fill="rgb(224,166,20)" rx="2" ry="2" />
<text  x="22.66" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (38 samples, 0.01%)</title><rect x="702.7" y="565" width="0.2" height="15.0" fill="rgb(246,9,28)" rx="2" ry="2" />
<text  x="705.72" y="575.5" ></text>
</g>
<g >
<title>runtime.futex (81 samples, 0.02%)</title><rect x="1005.8" y="757" width="0.3" height="15.0" fill="rgb(240,120,53)" rx="2" ry="2" />
<text  x="1008.79" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/syserror.TranslateError (287 samples, 0.09%)</title><rect x="217.7" y="773" width="1.0" height="15.0" fill="rgb(241,70,43)" rx="2" ry="2" />
<text  x="220.66" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress (468 samples, 0.14%)</title><rect x="1036.0" y="709" width="1.7" height="15.0" fill="rgb(224,186,44)" rx="2" ry="2" />
<text  x="1038.98" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,630 samples, 1.11%)</title><rect x="809.0" y="725" width="13.2" height="15.0" fill="rgb(232,29,28)" rx="2" ry="2" />
<text  x="812.03" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyInString (305 samples, 0.09%)</title><rect x="183.8" y="741" width="1.1" height="15.0" fill="rgb(205,175,4)" rx="2" ry="2" />
<text  x="186.79" y="751.5" ></text>
</g>
<g >
<title>runtime.startm (2,879 samples, 0.88%)</title><rect x="747.8" y="533" width="10.4" height="15.0" fill="rgb(233,102,46)" rx="2" ry="2" />
<text  x="750.77" y="543.5" ></text>
</g>
<g >
<title>runtime.mstart1 (47 samples, 0.01%)</title><rect x="16.8" y="821" width="0.1" height="15.0" fill="rgb(246,156,45)" rx="2" ry="2" />
<text  x="19.75" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (92 samples, 0.03%)</title><rect x="19.0" y="789" width="0.4" height="15.0" fill="rgb(207,47,22)" rx="2" ry="2" />
<text  x="22.03" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (182 samples, 0.06%)</title><rect x="1119.1" y="741" width="0.6" height="15.0" fill="rgb(238,132,30)" rx="2" ry="2" />
<text  x="1122.07" y="751.5" ></text>
</g>
<g >
<title>runtime.mallocgc (34 samples, 0.01%)</title><rect x="1092.6" y="645" width="0.1" height="15.0" fill="rgb(233,181,26)" rx="2" ry="2" />
<text  x="1095.60" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (54 samples, 0.02%)</title><rect x="18.5" y="629" width="0.2" height="15.0" fill="rgb(237,7,10)" rx="2" ry="2" />
<text  x="21.47" y="639.5" ></text>
</g>
<g >
<title>runtime.growslice (75 samples, 0.02%)</title><rect x="112.2" y="597" width="0.3" height="15.0" fill="rgb(208,176,45)" rx="2" ry="2" />
<text  x="115.20" y="607.5" ></text>
</g>
<g >
<title>all (326,293 samples, 100%)</title><rect x="10.0" y="917" width="1180.0" height="15.0" fill="rgb(247,196,16)" rx="2" ry="2" />
<text  x="13.00" y="927.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isResolutionRequiredRLocked (37 samples, 0.01%)</title><rect x="181.5" y="469" width="0.1" height="15.0" fill="rgb(233,227,14)" rx="2" ry="2" />
<text  x="184.50" y="479.5" ></text>
</g>
<g >
<title>runtime.binarySearchTree (46 samples, 0.01%)</title><rect x="1133.8" y="773" width="0.1" height="15.0" fill="rgb(207,85,26)" rx="2" ry="2" />
<text  x="1136.78" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Get (33 samples, 0.01%)</title><rect x="190.1" y="757" width="0.1" height="15.0" fill="rgb(228,84,7)" rx="2" ry="2" />
<text  x="193.07" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (140 samples, 0.04%)</title><rect x="115.4" y="485" width="0.5" height="15.0" fill="rgb(238,25,24)" rx="2" ry="2" />
<text  x="118.37" y="495.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="1128.4" y="581" width="0.2" height="15.0" fill="rgb(223,133,7)" rx="2" ry="2" />
<text  x="1131.37" y="591.5" ></text>
</g>
<g >
<title>runtime.memmove (42 samples, 0.01%)</title><rect x="184.1" y="645" width="0.2" height="15.0" fill="rgb(224,58,2)" rx="2" ry="2" />
<text  x="187.11" y="655.5" ></text>
</g>
<g >
<title>runtime.exitsyscallfast (228 samples, 0.07%)</title><rect x="449.7" y="741" width="0.8" height="15.0" fill="rgb(215,63,34)" rx="2" ry="2" />
<text  x="452.69" y="751.5" ></text>
</g>
<g >
<title>runtime.epollwait (370 samples, 0.11%)</title><rect x="114.5" y="629" width="1.4" height="15.0" fill="rgb(217,60,28)" rx="2" ry="2" />
<text  x="117.54" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (195 samples, 0.06%)</title><rect x="1186.9" y="677" width="0.7" height="15.0" fill="rgb(244,89,34)" rx="2" ry="2" />
<text  x="1189.91" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*TcpipTimer).Reset (686 samples, 0.21%)</title><rect x="1047.5" y="741" width="2.5" height="15.0" fill="rgb(235,203,39)" rx="2" ry="2" />
<text  x="1050.48" y="751.5" ></text>
</g>
<g >
<title>runtime.madvise (221 samples, 0.07%)</title><rect x="1130.1" y="789" width="0.8" height="15.0" fill="rgb(237,33,23)" rx="2" ry="2" />
<text  x="1133.10" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).executeSyscall (48,829 samples, 14.96%)</title><rect x="40.7" y="789" width="176.6" height="15.0" fill="rgb(240,226,36)" rx="2" ry="2" />
<text  x="43.74" y="799.5" >gvisor.dev/gvisor/pkg/..</text>
</g>
<g >
<title>runtime.bgscavenge (538 samples, 0.16%)</title><rect x="1107.5" y="869" width="1.9" height="15.0" fill="rgb(243,203,49)" rx="2" ry="2" />
<text  x="1110.46" y="879.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/secio.(*SectionWriter).Write (47 samples, 0.01%)</title><rect x="21.6" y="597" width="0.1" height="15.0" fill="rgb(220,210,37)" rx="2" ry="2" />
<text  x="24.57" y="607.5" ></text>
</g>
<g >
<title>runtime.newproc1 (79 samples, 0.02%)</title><rect x="1048.1" y="677" width="0.3" height="15.0" fill="rgb(241,11,8)" rx="2" ry="2" />
<text  x="1051.14" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*dispatcher).queuePacket (7,104 samples, 2.18%)</title><rect x="743.2" y="645" width="25.7" height="15.0" fill="rgb(248,149,27)" rx="2" ry="2" />
<text  x="746.20" y="655.5" >g..</text>
</g>
<g >
<title>runtime.memmove (50 samples, 0.02%)</title><rect x="206.1" y="549" width="0.2" height="15.0" fill="rgb(250,213,26)" rx="2" ry="2" />
<text  x="209.08" y="559.5" ></text>
</g>
<g >
<title>runtime.notewakeup (51 samples, 0.02%)</title><rect x="1109.1" y="741" width="0.2" height="15.0" fill="rgb(220,53,16)" rx="2" ry="2" />
<text  x="1112.12" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (172 samples, 0.05%)</title><rect x="1130.3" y="693" width="0.6" height="15.0" fill="rgb(240,161,8)" rx="2" ry="2" />
<text  x="1133.28" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/fsutil.(*inodeReadWriter).WriteFromBlocks (2,641 samples, 0.81%)</title><rect x="192.7" y="613" width="9.6" height="15.0" fill="rgb(238,93,23)" rx="2" ry="2" />
<text  x="195.70" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1185.9" y="693" width="0.1" height="15.0" fill="rgb(248,115,24)" rx="2" ry="2" />
<text  x="1188.90" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (58 samples, 0.02%)</title><rect x="637.8" y="613" width="0.2" height="15.0" fill="rgb(222,6,42)" rx="2" ry="2" />
<text  x="640.79" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).EventUnregister (32 samples, 0.01%)</title><rect x="173.3" y="757" width="0.1" height="15.0" fill="rgb(234,44,19)" rx="2" ry="2" />
<text  x="176.28" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (119 samples, 0.04%)</title><rect x="904.5" y="661" width="0.4" height="15.0" fill="rgb(225,35,8)" rx="2" ry="2" />
<text  x="907.47" y="671.5" ></text>
</g>
<g >
<title>runtime.futex (414 samples, 0.13%)</title><rect x="705.2" y="741" width="1.5" height="15.0" fill="rgb(231,126,16)" rx="2" ry="2" />
<text  x="708.20" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).LockUser (34 samples, 0.01%)</title><rect x="55.0" y="725" width="0.1" height="15.0" fill="rgb(219,171,15)" rx="2" ry="2" />
<text  x="58.00" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).enqueueAssertedWaker (62 samples, 0.02%)</title><rect x="1069.4" y="821" width="0.2" height="15.0" fill="rgb(249,103,10)" rx="2" ry="2" />
<text  x="1072.42" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (155 samples, 0.05%)</title><rect x="65.5" y="645" width="0.6" height="15.0" fill="rgb(248,205,44)" rx="2" ry="2" />
<text  x="68.53" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (797 samples, 0.24%)</title><rect x="635.1" y="645" width="2.9" height="15.0" fill="rgb(211,93,50)" rx="2" ry="2" />
<text  x="638.12" y="655.5" ></text>
</g>
<g >
<title>runtime.runqsteal (86 samples, 0.03%)</title><rect x="1185.7" y="805" width="0.3" height="15.0" fill="rgb(229,67,8)" rx="2" ry="2" />
<text  x="1188.69" y="815.5" ></text>
</g>
<g >
<title>runtime.handoffp (4,995 samples, 1.53%)</title><rect x="823.5" y="773" width="18.1" height="15.0" fill="rgb(246,147,26)" rx="2" ry="2" />
<text  x="826.55" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).IsBroadcast (38 samples, 0.01%)</title><rect x="784.6" y="677" width="0.2" height="15.0" fill="rgb(246,142,21)" rx="2" ry="2" />
<text  x="787.62" y="687.5" ></text>
</g>
<g >
<title>runtime.mallocgc (122 samples, 0.04%)</title><rect x="926.9" y="773" width="0.4" height="15.0" fill="rgb(236,139,37)" rx="2" ry="2" />
<text  x="929.87" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/fd.(*ReadWriter).WriteAt (47 samples, 0.01%)</title><rect x="21.6" y="581" width="0.1" height="15.0" fill="rgb(212,3,2)" rx="2" ry="2" />
<text  x="24.57" y="591.5" ></text>
</g>
<g >
<title>runtime.mallocgc (90 samples, 0.03%)</title><rect x="184.5" y="693" width="0.3" height="15.0" fill="rgb(220,135,11)" rx="2" ry="2" />
<text  x="187.47" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).nextWaker (103 samples, 0.03%)</title><rect x="1021.4" y="837" width="0.4" height="15.0" fill="rgb(235,185,37)" rx="2" ry="2" />
<text  x="1024.45" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (614 samples, 0.19%)</title><rect x="154.1" y="437" width="2.2" height="15.0" fill="rgb(239,155,37)" rx="2" ry="2" />
<text  x="157.07" y="447.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="695.5" y="613" width="0.1" height="15.0" fill="rgb(251,186,22)" rx="2" ry="2" />
<text  x="698.47" y="623.5" ></text>
</g>
<g >
<title>runtime.memmove (57 samples, 0.02%)</title><rect x="846.1" y="821" width="0.2" height="15.0" fill="rgb(249,47,11)" rx="2" ry="2" />
<text  x="849.13" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (138 samples, 0.04%)</title><rect x="183.9" y="693" width="0.5" height="15.0" fill="rgb(212,97,42)" rx="2" ry="2" />
<text  x="186.93" y="703.5" ></text>
</g>
<g >
<title>runtime.funcspdelta (74 samples, 0.02%)</title><rect x="1184.4" y="821" width="0.2" height="15.0" fill="rgb(239,152,29)" rx="2" ry="2" />
<text  x="1187.37" y="831.5" ></text>
</g>
<g >
<title>runtime.mallocgc (54 samples, 0.02%)</title><rect x="1031.2" y="725" width="0.2" height="15.0" fill="rgb(207,119,34)" rx="2" ry="2" />
<text  x="1034.18" y="735.5" ></text>
</g>
<g >
<title>runtime.futex (92 samples, 0.03%)</title><rect x="19.0" y="805" width="0.4" height="15.0" fill="rgb(239,32,32)" rx="2" ry="2" />
<text  x="22.03" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (46 samples, 0.01%)</title><rect x="847.9" y="693" width="0.2" height="15.0" fill="rgb(218,204,13)" rx="2" ry="2" />
<text  x="850.92" y="703.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (42 samples, 0.01%)</title><rect x="1042.7" y="741" width="0.1" height="15.0" fill="rgb(206,116,20)" rx="2" ry="2" />
<text  x="1045.68" y="751.5" ></text>
</g>
<g >
<title>__ext4_journal_get_write_access (28 samples, 0.01%)</title><rect x="200.8" y="213" width="0.1" height="15.0" fill="rgb(223,218,11)" rx="2" ry="2" />
<text  x="203.77" y="223.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (414 samples, 0.13%)</title><rect x="705.2" y="725" width="1.5" height="15.0" fill="rgb(234,21,25)" rx="2" ry="2" />
<text  x="708.20" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).SetTLS (95 samples, 0.03%)</title><rect x="223.1" y="821" width="0.4" height="15.0" fill="rgb(211,140,51)" rx="2" ry="2" />
<text  x="226.12" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (449 samples, 0.14%)</title><rect x="1125.2" y="661" width="1.6" height="15.0" fill="rgb(208,4,54)" rx="2" ry="2" />
<text  x="1128.17" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (119 samples, 0.04%)</title><rect x="1062.1" y="693" width="0.4" height="15.0" fill="rgb(225,144,41)" rx="2" ry="2" />
<text  x="1065.07" y="703.5" ></text>
</g>
<g >
<title>runtime.step (28 samples, 0.01%)</title><rect x="1184.5" y="789" width="0.1" height="15.0" fill="rgb(251,9,2)" rx="2" ry="2" />
<text  x="1187.52" y="799.5" ></text>
</g>
<g >
<title>runtime.send (200 samples, 0.06%)</title><rect x="1003.3" y="741" width="0.7" height="15.0" fill="rgb(226,117,15)" rx="2" ry="2" />
<text  x="1006.30" y="751.5" ></text>
</g>
<g >
<title>fmt.newPrinter (36 samples, 0.01%)</title><rect x="49.2" y="677" width="0.1" height="15.0" fill="rgb(234,83,31)" rx="2" ry="2" />
<text  x="52.18" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (359 samples, 0.11%)</title><rect x="920.6" y="613" width="1.3" height="15.0" fill="rgb(218,74,0)" rx="2" ry="2" />
<text  x="923.58" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,476 samples, 0.76%)</title><rect x="983.6" y="213" width="9.0" height="15.0" fill="rgb(231,80,35)" rx="2" ry="2" />
<text  x="986.64" y="223.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*segmentQueue).dequeue (43 samples, 0.01%)</title><rect x="1089.4" y="821" width="0.2" height="15.0" fill="rgb(231,7,11)" rx="2" ry="2" />
<text  x="1092.42" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.copyInPath (315 samples, 0.10%)</title><rect x="183.8" y="757" width="1.1" height="15.0" fill="rgb(205,109,52)" rx="2" ry="2" />
<text  x="186.79" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (87 samples, 0.03%)</title><rect x="1083.2" y="421" width="0.3" height="15.0" fill="rgb(229,180,51)" rx="2" ry="2" />
<text  x="1086.15" y="431.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).Value (40 samples, 0.01%)</title><rect x="47.6" y="693" width="0.1" height="15.0" fill="rgb(233,64,18)" rx="2" ry="2" />
<text  x="50.56" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Shutdown (1,066 samples, 0.33%)</title><rect x="179.6" y="741" width="3.9" height="15.0" fill="rgb(209,96,17)" rx="2" ry="2" />
<text  x="182.60" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (386 samples, 0.12%)</title><rect x="705.3" y="549" width="1.4" height="15.0" fill="rgb(229,189,39)" rx="2" ry="2" />
<text  x="708.30" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,852 samples, 1.49%)</title><rect x="877.3" y="693" width="17.6" height="15.0" fill="rgb(207,203,10)" rx="2" ry="2" />
<text  x="880.34" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.unrolledCalculateChecksum (34 samples, 0.01%)</title><rect x="1093.5" y="725" width="0.1" height="15.0" fill="rgb(233,188,17)" rx="2" ry="2" />
<text  x="1096.49" y="735.5" ></text>
</g>
<g >
<title>runtime.futex (189 samples, 0.06%)</title><rect x="702.2" y="725" width="0.7" height="15.0" fill="rgb(244,150,41)" rx="2" ry="2" />
<text  x="705.17" y="735.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (40 samples, 0.01%)</title><rect x="171.7" y="677" width="0.1" height="15.0" fill="rgb(212,213,26)" rx="2" ry="2" />
<text  x="174.68" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (102 samples, 0.03%)</title><rect x="80.7" y="661" width="0.3" height="15.0" fill="rgb(246,77,31)" rx="2" ry="2" />
<text  x="83.66" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*listenContext).addPendingEndpoint (78 samples, 0.02%)</title><rect x="1031.9" y="805" width="0.3" height="15.0" fill="rgb(235,202,39)" rx="2" ry="2" />
<text  x="1034.90" y="815.5" ></text>
</g>
<g >
<title>runtime.runqgrab (677 samples, 0.21%)</title><rect x="1120.7" y="789" width="2.5" height="15.0" fill="rgb(241,222,29)" rx="2" ry="2" />
<text  x="1123.70" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (103 samples, 0.03%)</title><rect x="69.7" y="709" width="0.4" height="15.0" fill="rgb(230,105,9)" rx="2" ry="2" />
<text  x="72.73" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (234 samples, 0.07%)</title><rect x="704.0" y="597" width="0.9" height="15.0" fill="rgb(249,195,12)" rx="2" ry="2" />
<text  x="707.03" y="607.5" ></text>
</g>
<g >
<title>runtime.usleep (67 samples, 0.02%)</title><rect x="22.4" y="773" width="0.2" height="15.0" fill="rgb(217,16,48)" rx="2" ry="2" />
<text  x="25.37" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/iovec.(*Builder).Add (32 samples, 0.01%)</title><rect x="923.3" y="821" width="0.1" height="15.0" fill="rgb(230,32,20)" rx="2" ry="2" />
<text  x="926.30" y="831.5" ></text>
</g>
<g >
<title>runtime.mallocgc (36 samples, 0.01%)</title><rect x="1093.9" y="741" width="0.1" height="15.0" fill="rgb(217,135,17)" rx="2" ry="2" />
<text  x="1096.87" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.FromIOWriter.writeFromBlock (2,375 samples, 0.73%)</title><rect x="193.2" y="549" width="8.6" height="15.0" fill="rgb(233,204,4)" rx="2" ry="2" />
<text  x="196.24" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*endpoint).sendBatch (20,652 samples, 6.33%)</title><rect x="923.5" y="821" width="74.6" height="15.0" fill="rgb(254,202,45)" rx="2" ry="2" />
<text  x="926.45" y="831.5" >gvisor.d..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (237 samples, 0.07%)</title><rect x="15.0" y="629" width="0.9" height="15.0" fill="rgb(213,105,45)" rx="2" ry="2" />
<text  x="18.01" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (69 samples, 0.02%)</title><rect x="1007.4" y="597" width="0.3" height="15.0" fill="rgb(211,13,36)" rx="2" ry="2" />
<text  x="1010.41" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (293 samples, 0.09%)</title><rect x="199.8" y="293" width="1.1" height="15.0" fill="rgb(220,47,20)" rx="2" ry="2" />
<text  x="202.82" y="303.5" ></text>
</g>
<g >
<title>runtime.mcall (165 samples, 0.05%)</title><rect x="451.6" y="741" width="0.6" height="15.0" fill="rgb(219,161,28)" rx="2" ry="2" />
<text  x="454.64" y="751.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (45 samples, 0.01%)</title><rect x="704.9" y="805" width="0.1" height="15.0" fill="rgb(230,179,1)" rx="2" ry="2" />
<text  x="707.88" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (171 samples, 0.05%)</title><rect x="894.3" y="629" width="0.6" height="15.0" fill="rgb(212,8,40)" rx="2" ry="2" />
<text  x="897.27" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (38 samples, 0.01%)</title><rect x="1078.6" y="677" width="0.2" height="15.0" fill="rgb(214,194,36)" rx="2" ry="2" />
<text  x="1081.64" y="687.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (167 samples, 0.05%)</title><rect x="781.2" y="725" width="0.6" height="15.0" fill="rgb(228,53,50)" rx="2" ry="2" />
<text  x="784.23" y="735.5" ></text>
</g>
<g >
<title>runtime.systemstack (112 samples, 0.03%)</title><rect x="65.6" y="613" width="0.4" height="15.0" fill="rgb(253,35,45)" rx="2" ry="2" />
<text  x="68.59" y="623.5" ></text>
</g>
<g >
<title>runtime.systemstack (46 samples, 0.01%)</title><rect x="767.9" y="549" width="0.2" height="15.0" fill="rgb(215,163,48)" rx="2" ry="2" />
<text  x="770.93" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (73 samples, 0.02%)</title><rect x="1108.8" y="597" width="0.2" height="15.0" fill="rgb(223,155,38)" rx="2" ry="2" />
<text  x="1111.77" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/syserr.TranslateNetstackError (32 samples, 0.01%)</title><rect x="175.5" y="709" width="0.1" height="15.0" fill="rgb(216,80,5)" rx="2" ry="2" />
<text  x="178.50" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).IsLoopback (31 samples, 0.01%)</title><rect x="791.4" y="709" width="0.1" height="15.0" fill="rgb(216,122,4)" rx="2" ry="2" />
<text  x="794.40" y="719.5" ></text>
</g>
<g >
<title>runtime.mcall (32 samples, 0.01%)</title><rect x="16.5" y="837" width="0.1" height="15.0" fill="rgb(242,14,33)" rx="2" ry="2" />
<text  x="19.50" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*TcpipTimer).Reset (1,345 samples, 0.41%)</title><rect x="1078.9" y="677" width="4.9" height="15.0" fill="rgb(251,86,13)" rx="2" ry="2" />
<text  x="1081.94" y="687.5" ></text>
</g>
<g >
<title>runtime.gentraceback (879 samples, 0.27%)</title><rect x="1052.1" y="725" width="3.2" height="15.0" fill="rgb(217,195,10)" rx="2" ry="2" />
<text  x="1055.11" y="735.5" ></text>
</g>
<g >
<title>runtime.runqempty (38 samples, 0.01%)</title><rect x="1066.5" y="757" width="0.2" height="15.0" fill="rgb(225,194,36)" rx="2" ry="2" />
<text  x="1069.54" y="767.5" ></text>
</g>
<g >
<title>runtime.mcall (85 samples, 0.03%)</title><rect x="1111.2" y="853" width="0.4" height="15.0" fill="rgb(227,135,20)" rx="2" ry="2" />
<text  x="1114.25" y="863.5" ></text>
</g>
<g >
<title>runtime.startm (62 samples, 0.02%)</title><rect x="65.8" y="549" width="0.2" height="15.0" fill="rgb(241,6,14)" rx="2" ry="2" />
<text  x="68.75" y="559.5" ></text>
</g>
<g >
<title>runtime.convI2I (59 samples, 0.02%)</title><rect x="1035.6" y="725" width="0.2" height="15.0" fill="rgb(235,145,10)" rx="2" ry="2" />
<text  x="1038.63" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*inodeFileState).WriteFromBlocksAt (47 samples, 0.01%)</title><rect x="21.6" y="661" width="0.1" height="15.0" fill="rgb(249,151,53)" rx="2" ry="2" />
<text  x="24.57" y="671.5" ></text>
</g>
<g >
<title>runtime.(*itabTableType).find (104 samples, 0.03%)</title><rect x="58.0" y="725" width="0.4" height="15.0" fill="rgb(222,30,25)" rx="2" ry="2" />
<text  x="61.03" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (87 samples, 0.03%)</title><rect x="117.8" y="597" width="0.3" height="15.0" fill="rgb(223,9,36)" rx="2" ry="2" />
<text  x="120.77" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*fileOperations).Write (3,058 samples, 0.94%)</title><rect x="191.9" y="709" width="11.0" height="15.0" fill="rgb(235,165,32)" rx="2" ry="2" />
<text  x="194.86" y="719.5" ></text>
</g>
<g >
<title>runtime.mstart (47 samples, 0.01%)</title><rect x="16.8" y="837" width="0.1" height="15.0" fill="rgb(254,100,12)" rx="2" ry="2" />
<text  x="19.75" y="847.5" ></text>
</g>
<g >
<title>runtime.selectgo (1,667 samples, 0.51%)</title><rect x="697.7" y="853" width="6.0" height="15.0" fill="rgb(219,20,54)" rx="2" ry="2" />
<text  x="700.67" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendSynTCP (1,475 samples, 0.45%)</title><rect x="1026.4" y="789" width="5.3" height="15.0" fill="rgb(231,24,4)" rx="2" ry="2" />
<text  x="1029.40" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Remove (159 samples, 0.05%)</title><rect x="68.9" y="757" width="0.6" height="15.0" fill="rgb(230,0,7)" rx="2" ry="2" />
<text  x="71.93" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn.func1 (52 samples, 0.02%)</title><rect x="204.0" y="677" width="0.2" height="15.0" fill="rgb(224,143,47)" rx="2" ry="2" />
<text  x="206.96" y="687.5" ></text>
</g>
<g >
<title>runtime.acquirep (29 samples, 0.01%)</title><rect x="1123.3" y="789" width="0.1" height="15.0" fill="rgb(249,85,19)" rx="2" ry="2" />
<text  x="1126.28" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (97 samples, 0.03%)</title><rect x="702.5" y="629" width="0.4" height="15.0" fill="rgb(240,38,7)" rx="2" ry="2" />
<text  x="705.50" y="639.5" ></text>
</g>
<g >
<title>runtime.lock2 (58 samples, 0.02%)</title><rect x="1003.1" y="741" width="0.2" height="15.0" fill="rgb(228,40,19)" rx="2" ry="2" />
<text  x="1006.08" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (152 samples, 0.05%)</title><rect x="840.7" y="533" width="0.5" height="15.0" fill="rgb(212,33,38)" rx="2" ry="2" />
<text  x="843.70" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut (32 samples, 0.01%)</title><rect x="57.5" y="709" width="0.1" height="15.0" fill="rgb(208,134,5)" rx="2" ry="2" />
<text  x="60.45" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.AddDanglingEndpoint (136 samples, 0.04%)</title><rect x="65.0" y="661" width="0.5" height="15.0" fill="rgb(251,18,47)" rx="2" ry="2" />
<text  x="68.01" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).initialReceiveWindow (89 samples, 0.03%)</title><rect x="1024.4" y="773" width="0.3" height="15.0" fill="rgb(225,96,33)" rx="2" ry="2" />
<text  x="1027.41" y="783.5" ></text>
</g>
<g >
<title>nf_conntrack_in (1,278 samples, 0.39%)</title><rect x="959.9" y="469" width="4.6" height="15.0" fill="rgb(241,137,37)" rx="2" ry="2" />
<text  x="962.86" y="479.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (939 samples, 0.29%)</title><rect x="907.4" y="693" width="3.4" height="15.0" fill="rgb(236,93,49)" rx="2" ry="2" />
<text  x="910.38" y="703.5" ></text>
</g>
<g >
<title>runtime.notewakeup (84 samples, 0.03%)</title><rect x="1005.8" y="773" width="0.3" height="15.0" fill="rgb(213,17,7)" rx="2" ry="2" />
<text  x="1008.78" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/rawfile.callExitsyscall (855 samples, 0.26%)</title><rect x="842.0" y="821" width="3.1" height="15.0" fill="rgb(233,50,18)" rx="2" ry="2" />
<text  x="845.02" y="831.5" ></text>
</g>
<g >
<title>runtime.checkTimers (28 samples, 0.01%)</title><rect x="22.0" y="805" width="0.1" height="15.0" fill="rgb(241,228,4)" rx="2" ry="2" />
<text  x="25.02" y="815.5" ></text>
</g>
<g >
<title>runtime.notewakeup (4,738 samples, 1.45%)</title><rect x="824.2" y="741" width="17.1" height="15.0" fill="rgb(205,87,41)" rx="2" ry="2" />
<text  x="827.16" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,639 samples, 0.50%)</title><rect x="752.1" y="437" width="6.0" height="15.0" fill="rgb(241,226,24)" rx="2" ry="2" />
<text  x="755.14" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).ConfirmReachable (2,529 samples, 0.78%)</title><rect x="1078.1" y="789" width="9.1" height="15.0" fill="rgb(241,93,26)" rx="2" ry="2" />
<text  x="1081.09" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (151 samples, 0.05%)</title><rect x="451.7" y="517" width="0.5" height="15.0" fill="rgb(236,132,26)" rx="2" ry="2" />
<text  x="454.69" y="527.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.calculateAdvertisedMSS (50 samples, 0.02%)</title><rect x="1039.3" y="789" width="0.1" height="15.0" fill="rgb(233,177,9)" rx="2" ry="2" />
<text  x="1042.26" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (281 samples, 0.09%)</title><rect x="155.3" y="405" width="1.0" height="15.0" fill="rgb(221,96,32)" rx="2" ry="2" />
<text  x="158.28" y="415.5" ></text>
</g>
<g >
<title>runtime.makechan (206 samples, 0.06%)</title><rect x="171.1" y="741" width="0.8" height="15.0" fill="rgb(245,171,17)" rx="2" ry="2" />
<text  x="174.14" y="751.5" ></text>
</g>
<g >
<title>runtime.mallocgc (31 samples, 0.01%)</title><rect x="46.2" y="693" width="0.1" height="15.0" fill="rgb(236,54,24)" rx="2" ry="2" />
<text  x="49.16" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (70 samples, 0.02%)</title><rect x="1152.9" y="613" width="0.2" height="15.0" fill="rgb(245,103,32)" rx="2" ry="2" />
<text  x="1155.85" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (141 samples, 0.04%)</title><rect x="451.7" y="469" width="0.5" height="15.0" fill="rgb(244,41,21)" rx="2" ry="2" />
<text  x="454.73" y="479.5" ></text>
</g>
<g >
<title>runtime.newproc.func1 (76 samples, 0.02%)</title><rect x="114.1" y="549" width="0.3" height="15.0" fill="rgb(215,209,34)" rx="2" ry="2" />
<text  x="117.13" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (59 samples, 0.02%)</title><rect x="1189.8" y="581" width="0.2" height="15.0" fill="rgb(223,119,44)" rx="2" ry="2" />
<text  x="1192.78" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (89 samples, 0.03%)</title><rect x="149.0" y="581" width="0.4" height="15.0" fill="rgb(251,140,32)" rx="2" ry="2" />
<text  x="152.03" y="591.5" ></text>
</g>
<g >
<title>runtime.resetspinning (570 samples, 0.17%)</title><rect x="1187.7" y="821" width="2.0" height="15.0" fill="rgb(246,16,9)" rx="2" ry="2" />
<text  x="1190.66" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn.func1 (88 samples, 0.03%)</title><rect x="205.9" y="581" width="0.4" height="15.0" fill="rgb(237,57,31)" rx="2" ry="2" />
<text  x="208.95" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).CooperativelySchedulesAddressSpace (44 samples, 0.01%)</title><rect x="84.4" y="677" width="0.2" height="15.0" fill="rgb(245,27,20)" rx="2" ry="2" />
<text  x="87.41" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).DecRef (39 samples, 0.01%)</title><rect x="185.3" y="709" width="0.1" height="15.0" fill="rgb(207,94,35)" rx="2" ry="2" />
<text  x="188.28" y="719.5" ></text>
</g>
<g >
<title>runtime.makeslice (43 samples, 0.01%)</title><rect x="1093.9" y="757" width="0.1" height="15.0" fill="rgb(247,200,28)" rx="2" ry="2" />
<text  x="1096.85" y="767.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (342 samples, 0.10%)</title><rect x="847.4" y="789" width="1.3" height="15.0" fill="rgb(250,5,14)" rx="2" ry="2" />
<text  x="850.42" y="799.5" ></text>
</g>
<g >
<title>runtime.(*mheap).alloc.func1 (99 samples, 0.03%)</title><rect x="848.1" y="709" width="0.4" height="15.0" fill="rgb(245,89,50)" rx="2" ry="2" />
<text  x="851.09" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).dumpPacket (82 samples, 0.03%)</title><rect x="795.9" y="773" width="0.3" height="15.0" fill="rgb(254,33,17)" rx="2" ry="2" />
<text  x="798.95" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (270 samples, 0.08%)</title><rect x="703.9" y="677" width="1.0" height="15.0" fill="rgb(242,77,51)" rx="2" ry="2" />
<text  x="706.90" y="687.5" ></text>
</g>
<g >
<title>runtime.usleep (121 samples, 0.04%)</title><rect x="1062.1" y="709" width="0.4" height="15.0" fill="rgb(236,149,6)" rx="2" ry="2" />
<text  x="1065.06" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).DeliverTransportPacket (92 samples, 0.03%)</title><rect x="19.7" y="805" width="0.3" height="15.0" fill="rgb(227,217,48)" rx="2" ry="2" />
<text  x="22.66" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (28 samples, 0.01%)</title><rect x="63.0" y="693" width="0.1" height="15.0" fill="rgb(227,110,2)" rx="2" ry="2" />
<text  x="66.02" y="703.5" ></text>
</g>
<g >
<title>runtime.mapiternext (88 samples, 0.03%)</title><rect x="738.2" y="677" width="0.4" height="15.0" fill="rgb(216,170,21)" rx="2" ry="2" />
<text  x="741.24" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (182 samples, 0.06%)</title><rect x="212.5" y="149" width="0.6" height="15.0" fill="rgb(233,77,43)" rx="2" ry="2" />
<text  x="215.46" y="159.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).get (74 samples, 0.02%)</title><rect x="1092.3" y="661" width="0.2" height="15.0" fill="rgb(254,207,9)" rx="2" ry="2" />
<text  x="1095.26" y="671.5" ></text>
</g>
<g >
<title>runtime.pcdatavalue (55 samples, 0.02%)</title><rect x="1184.1" y="789" width="0.2" height="15.0" fill="rgb(215,172,2)" rx="2" ry="2" />
<text  x="1187.07" y="799.5" ></text>
</g>
<g >
<title>[[vdso]] (185 samples, 0.06%)</title><rect x="30.8" y="837" width="0.6" height="15.0" fill="rgb(236,151,19)" rx="2" ry="2" />
<text  x="33.77" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportDemuxer).findTransportEndpoint (184 samples, 0.06%)</title><rect x="1067.3" y="821" width="0.7" height="15.0" fill="rgb(245,83,15)" rx="2" ry="2" />
<text  x="1070.35" y="831.5" ></text>
</g>
<g >
<title>runtime.findObject (70 samples, 0.02%)</title><rect x="1132.0" y="821" width="0.3" height="15.0" fill="rgb(225,205,25)" rx="2" ry="2" />
<text  x="1135.00" y="831.5" ></text>
</g>
<g >
<title>runtime.runqempty (38 samples, 0.01%)</title><rect x="921.9" y="757" width="0.1" height="15.0" fill="rgb(234,117,16)" rx="2" ry="2" />
<text  x="924.91" y="767.5" ></text>
</g>
<g >
<title>runtime.interequal (32 samples, 0.01%)</title><rect x="217.8" y="757" width="0.1" height="15.0" fill="rgb(250,73,13)" rx="2" ry="2" />
<text  x="220.75" y="767.5" ></text>
</g>
<g >
<title>runtime.pcvalue (114 samples, 0.03%)</title><rect x="1103.3" y="661" width="0.4" height="15.0" fill="rgb(206,87,31)" rx="2" ry="2" />
<text  x="1106.33" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).IsLoopback (62 samples, 0.02%)</title><rect x="790.6" y="693" width="0.2" height="15.0" fill="rgb(240,4,2)" rx="2" ry="2" />
<text  x="793.59" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).Forwarding (38 samples, 0.01%)</title><rect x="1034.4" y="773" width="0.1" height="15.0" fill="rgb(222,195,32)" rx="2" ry="2" />
<text  x="1037.39" y="783.5" ></text>
</g>
<g >
<title>runtime.step (30 samples, 0.01%)</title><rect x="1184.1" y="757" width="0.2" height="15.0" fill="rgb(239,89,54)" rx="2" ry="2" />
<text  x="1187.15" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).isValidForOutgoing (61 samples, 0.02%)</title><rect x="1075.6" y="661" width="0.2" height="15.0" fill="rgb(205,186,53)" rx="2" ry="2" />
<text  x="1078.59" y="671.5" ></text>
</g>
<g >
<title>runtime.newstack (1,293 samples, 0.40%)</title><rect x="1094.6" y="773" width="4.7" height="15.0" fill="rgb(240,92,42)" rx="2" ry="2" />
<text  x="1097.58" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).IsBroadcast (172 samples, 0.05%)</title><rect x="728.7" y="709" width="0.7" height="15.0" fill="rgb(254,17,8)" rx="2" ry="2" />
<text  x="731.74" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress (2,381 samples, 0.73%)</title><rect x="782.6" y="709" width="8.6" height="15.0" fill="rgb(233,219,30)" rx="2" ry="2" />
<text  x="785.60" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (432 samples, 0.13%)</title><rect x="196.3" y="293" width="1.5" height="15.0" fill="rgb(226,33,43)" rx="2" ry="2" />
<text  x="199.25" y="303.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (29 samples, 0.01%)</title><rect x="1127.0" y="789" width="0.1" height="15.0" fill="rgb(231,206,22)" rx="2" ry="2" />
<text  x="1130.02" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (32 samples, 0.01%)</title><rect x="782.2" y="725" width="0.1" height="15.0" fill="rgb(226,108,19)" rx="2" ry="2" />
<text  x="785.23" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (143 samples, 0.04%)</title><rect x="781.3" y="709" width="0.5" height="15.0" fill="rgb(217,9,31)" rx="2" ry="2" />
<text  x="784.32" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (12,983 samples, 3.98%)</title><rect x="949.7" y="565" width="47.0" height="15.0" fill="rgb(205,17,45)" rx="2" ry="2" />
<text  x="952.75" y="575.5" >[[ke..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).UnstableAttr (58 samples, 0.02%)</title><rect x="191.2" y="677" width="0.2" height="15.0" fill="rgb(208,212,5)" rx="2" ry="2" />
<text  x="194.22" y="687.5" ></text>
</g>
<g >
<title>runtime.findfunc (29 samples, 0.01%)</title><rect x="1080.6" y="581" width="0.1" height="15.0" fill="rgb(239,19,12)" rx="2" ry="2" />
<text  x="1083.59" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/buffer.(*VectorisedView).TrimFront (29 samples, 0.01%)</title><rect x="793.9" y="693" width="0.1" height="15.0" fill="rgb(219,205,34)" rx="2" ry="2" />
<text  x="796.89" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MainAddress (690 samples, 0.21%)</title><rect x="724.7" y="725" width="2.5" height="15.0" fill="rgb(242,89,3)" rx="2" ry="2" />
<text  x="727.67" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).accountTaskGoroutineLeave (252 samples, 0.08%)</title><rect x="37.3" y="837" width="1.0" height="15.0" fill="rgb(227,191,9)" rx="2" ry="2" />
<text  x="40.34" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (176 samples, 0.05%)</title><rect x="996.0" y="421" width="0.6" height="15.0" fill="rgb(249,47,12)" rx="2" ry="2" />
<text  x="999.00" y="431.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (282 samples, 0.09%)</title><rect x="14.8" y="661" width="1.1" height="15.0" fill="rgb(215,155,53)" rx="2" ry="2" />
<text  x="17.85" y="671.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (162 samples, 0.05%)</title><rect x="1028.2" y="533" width="0.6" height="15.0" fill="rgb(220,45,49)" rx="2" ry="2" />
<text  x="1031.20" y="543.5" ></text>
</g>
<g >
<title>runtime.(*mheap).nextSpanForSweep (55 samples, 0.02%)</title><rect x="1109.9" y="837" width="0.2" height="15.0" fill="rgb(253,229,9)" rx="2" ry="2" />
<text  x="1112.87" y="847.5" ></text>
</g>
<g >
<title>runtime.mallocgc (98 samples, 0.03%)</title><rect x="762.1" y="581" width="0.4" height="15.0" fill="rgb(221,223,26)" rx="2" ry="2" />
<text  x="765.12" y="591.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (3,239 samples, 0.99%)</title><rect x="746.5" y="581" width="11.7" height="15.0" fill="rgb(249,92,24)" rx="2" ry="2" />
<text  x="749.52" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (59 samples, 0.02%)</title><rect x="1122.9" y="597" width="0.2" height="15.0" fill="rgb(234,195,7)" rx="2" ry="2" />
<text  x="1125.93" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (59 samples, 0.02%)</title><rect x="1152.9" y="597" width="0.2" height="15.0" fill="rgb(249,13,23)" rx="2" ry="2" />
<text  x="1155.89" y="607.5" ></text>
</g>
<g >
<title>runtime.stopm (158 samples, 0.05%)</title><rect x="1108.5" y="773" width="0.5" height="15.0" fill="rgb(232,86,37)" rx="2" ry="2" />
<text  x="1111.47" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (2,753 samples, 0.84%)</title><rect x="192.6" y="645" width="10.0" height="15.0" fill="rgb(224,97,23)" rx="2" ry="2" />
<text  x="195.60" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (168 samples, 0.05%)</title><rect x="1187.0" y="645" width="0.6" height="15.0" fill="rgb(218,60,13)" rx="2" ry="2" />
<text  x="1190.01" y="655.5" ></text>
</g>
<g >
<title>runtime.acquireSudog (48 samples, 0.01%)</title><rect x="698.5" y="837" width="0.2" height="15.0" fill="rgb(231,69,17)" rx="2" ry="2" />
<text  x="701.52" y="847.5" ></text>
</g>
<g >
<title>runtime.memhash64 (195 samples, 0.06%)</title><rect x="456.3" y="805" width="0.7" height="15.0" fill="rgb(215,90,25)" rx="2" ry="2" />
<text  x="459.34" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (47 samples, 0.01%)</title><rect x="21.6" y="709" width="0.1" height="15.0" fill="rgb(217,197,35)" rx="2" ry="2" />
<text  x="24.57" y="719.5" ></text>
</g>
<g >
<title>__nf_conntrack_find_get (45 samples, 0.01%)</title><rect x="991.3" y="165" width="0.2" height="15.0" fill="rgb(211,39,54)" rx="2" ry="2" />
<text  x="994.34" y="175.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).nextWaker (353 samples, 0.11%)</title><rect x="1006.5" y="821" width="1.2" height="15.0" fill="rgb(237,1,2)" rx="2" ry="2" />
<text  x="1009.47" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).getAddressOrCreateTempInner (356 samples, 0.11%)</title><rect x="1034.6" y="741" width="1.3" height="15.0" fill="rgb(250,123,18)" rx="2" ry="2" />
<text  x="1037.64" y="751.5" ></text>
</g>
<g >
<title>runtime.mallocgc (741 samples, 0.23%)</title><rect x="846.6" y="805" width="2.7" height="15.0" fill="rgb(248,97,31)" rx="2" ry="2" />
<text  x="849.58" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="990.7" y="37" width="0.1" height="15.0" fill="rgb(222,26,22)" rx="2" ry="2" />
<text  x="993.69" y="47.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).getAddressOrCreateTempInner (2,816 samples, 0.86%)</title><rect x="782.5" y="725" width="10.2" height="15.0" fill="rgb(223,19,52)" rx="2" ry="2" />
<text  x="785.47" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (124 samples, 0.04%)</title><rect x="157.4" y="581" width="0.4" height="15.0" fill="rgb(210,44,22)" rx="2" ry="2" />
<text  x="160.37" y="591.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (34 samples, 0.01%)</title><rect x="930.0" y="757" width="0.1" height="15.0" fill="rgb(236,23,1)" rx="2" ry="2" />
<text  x="932.96" y="767.5" ></text>
</g>
<g >
<title>runtime.findrunnable (56 samples, 0.02%)</title><rect x="1111.3" y="805" width="0.2" height="15.0" fill="rgb(217,158,27)" rx="2" ry="2" />
<text  x="1114.29" y="815.5" ></text>
</g>
<g >
<title>runtime.step (31 samples, 0.01%)</title><rect x="1137.3" y="741" width="0.1" height="15.0" fill="rgb(253,78,41)" rx="2" ry="2" />
<text  x="1140.33" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*AtomicRefCount).DecRefWithDestructor (32 samples, 0.01%)</title><rect x="78.4" y="725" width="0.1" height="15.0" fill="rgb(225,202,13)" rx="2" ry="2" />
<text  x="81.39" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (156 samples, 0.05%)</title><rect x="451.7" y="581" width="0.5" height="15.0" fill="rgb(220,206,9)" rx="2" ry="2" />
<text  x="454.67" y="591.5" ></text>
</g>
<g >
<title>ext4_xattr_get (223 samples, 0.07%)</title><rect x="197.0" y="229" width="0.8" height="15.0" fill="rgb(252,0,49)" rx="2" ry="2" />
<text  x="199.98" y="239.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).AcquireAssignedAddressOrMatching (398 samples, 0.12%)</title><rect x="1036.1" y="693" width="1.4" height="15.0" fill="rgb(234,205,23)" rx="2" ry="2" />
<text  x="1039.10" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (82 samples, 0.03%)</title><rect x="13.1" y="613" width="0.3" height="15.0" fill="rgb(253,227,30)" rx="2" ry="2" />
<text  x="16.10" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).getAddressEP (80 samples, 0.02%)</title><rect x="1032.8" y="757" width="0.3" height="15.0" fill="rgb(252,165,44)" rx="2" ry="2" />
<text  x="1035.82" y="767.5" ></text>
</g>
<g >
<title>runtime.pcdatavalue (676 samples, 0.21%)</title><rect x="1139.8" y="709" width="2.5" height="15.0" fill="rgb(219,188,0)" rx="2" ry="2" />
<text  x="1142.84" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).getAddressOrCreateTempInner (63 samples, 0.02%)</title><rect x="1032.9" y="725" width="0.2" height="15.0" fill="rgb(208,157,23)" rx="2" ry="2" />
<text  x="1035.88" y="735.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast32 (280 samples, 0.09%)</title><rect x="452.7" y="789" width="1.0" height="15.0" fill="rgb(222,4,53)" rx="2" ry="2" />
<text  x="455.71" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (708 samples, 0.22%)</title><rect x="838.7" y="549" width="2.5" height="15.0" fill="rgb(233,192,40)" rx="2" ry="2" />
<text  x="841.69" y="559.5" ></text>
</g>
<g >
<title>runtime.dodeltimer (42 samples, 0.01%)</title><rect x="112.0" y="597" width="0.2" height="15.0" fill="rgb(234,57,17)" rx="2" ry="2" />
<text  x="115.04" y="607.5" ></text>
</g>
<g >
<title>runtime.futex (76 samples, 0.02%)</title><rect x="19.7" y="597" width="0.2" height="15.0" fill="rgb(230,180,29)" rx="2" ry="2" />
<text  x="22.66" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportDemuxer).deliverRawPacket (357 samples, 0.11%)</title><rect x="773.9" y="693" width="1.2" height="15.0" fill="rgb(238,186,25)" rx="2" ry="2" />
<text  x="776.86" y="703.5" ></text>
</g>
<g >
<title>runtime.newobject (63 samples, 0.02%)</title><rect x="1081.6" y="629" width="0.2" height="15.0" fill="rgb(230,89,50)" rx="2" ry="2" />
<text  x="1084.57" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (52 samples, 0.02%)</title><rect x="726.4" y="645" width="0.2" height="15.0" fill="rgb(216,30,51)" rx="2" ry="2" />
<text  x="729.38" y="655.5" ></text>
</g>
<g >
<title>runtime.resetspinning (379 samples, 0.12%)</title><rect x="1127.2" y="821" width="1.4" height="15.0" fill="rgb(221,95,34)" rx="2" ry="2" />
<text  x="1130.20" y="831.5" ></text>
</g>
<g >
<title>runtime.adjustframe (96 samples, 0.03%)</title><rect x="1183.9" y="821" width="0.4" height="15.0" fill="rgb(230,85,4)" rx="2" ry="2" />
<text  x="1186.93" y="831.5" ></text>
</g>
<g >
<title>time.goFunc (91 samples, 0.03%)</title><rect x="114.1" y="581" width="0.3" height="15.0" fill="rgb(246,24,33)" rx="2" ry="2" />
<text  x="117.09" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*protocol).ParsePorts (29 samples, 0.01%)</title><rect x="777.2" y="709" width="0.1" height="15.0" fill="rgb(237,71,23)" rx="2" ry="2" />
<text  x="780.20" y="719.5" ></text>
</g>
<g >
<title>runtime.wakep (766 samples, 0.23%)</title><rect x="919.1" y="741" width="2.8" height="15.0" fill="rgb(208,131,8)" rx="2" ry="2" />
<text  x="922.14" y="751.5" ></text>
</g>
<g >
<title>runtime.selectgo (44 samples, 0.01%)</title><rect x="696.8" y="853" width="0.2" height="15.0" fill="rgb(219,73,37)" rx="2" ry="2" />
<text  x="699.81" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (65 samples, 0.02%)</title><rect x="766.3" y="469" width="0.2" height="15.0" fill="rgb(221,3,46)" rx="2" ry="2" />
<text  x="769.25" y="479.5" ></text>
</g>
<g >
<title>runtime.mapiternext (38 samples, 0.01%)</title><rect x="1037.3" y="661" width="0.1" height="15.0" fill="rgb(249,157,10)" rx="2" ry="2" />
<text  x="1040.26" y="671.5" ></text>
</g>
<g >
<title>br_fdb_update (243 samples, 0.07%)</title><rect x="966.3" y="437" width="0.9" height="15.0" fill="rgb(229,218,46)" rx="2" ry="2" />
<text  x="969.34" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/limits.(*LimitSet).Get (48 samples, 0.01%)</title><rect x="190.9" y="709" width="0.2" height="15.0" fill="rgb(209,162,13)" rx="2" ry="2" />
<text  x="193.93" y="719.5" ></text>
</g>
<g >
<title>runtime.makeslice (186 samples, 0.06%)</title><rect x="926.7" y="789" width="0.7" height="15.0" fill="rgb(235,184,19)" rx="2" ry="2" />
<text  x="929.72" y="799.5" ></text>
</g>
<g >
<title>runtime.schedule (206 samples, 0.06%)</title><rect x="1045.8" y="789" width="0.8" height="15.0" fill="rgb(210,101,41)" rx="2" ry="2" />
<text  x="1048.83" y="799.5" ></text>
</g>
<g >
<title>runtime.notetsleep_internal (147 samples, 0.05%)</title><rect x="12.9" y="741" width="0.5" height="15.0" fill="rgb(209,125,13)" rx="2" ry="2" />
<text  x="15.86" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.sendTCP (688 samples, 0.21%)</title><rect x="180.4" y="581" width="2.5" height="15.0" fill="rgb(236,151,20)" rx="2" ry="2" />
<text  x="183.39" y="591.5" ></text>
</g>
<g >
<title>runtime.mcall (6,954 samples, 2.13%)</title><rect x="897.2" y="805" width="25.1" height="15.0" fill="rgb(227,8,16)" rx="2" ry="2" />
<text  x="900.16" y="815.5" >r..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).nextWaker (2,857 samples, 0.88%)</title><rect x="1056.6" y="821" width="10.3" height="15.0" fill="rgb(207,104,52)" rx="2" ry="2" />
<text  x="1059.59" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).DeliverOutboundPacket (44 samples, 0.01%)</title><rect x="210.2" y="421" width="0.2" height="15.0" fill="rgb(221,91,1)" rx="2" ry="2" />
<text  x="213.21" y="431.5" ></text>
</g>
<g >
<title>runtime.newobject (48 samples, 0.01%)</title><rect x="1092.6" y="661" width="0.1" height="15.0" fill="rgb(230,177,52)" rx="2" ry="2" />
<text  x="1095.56" y="671.5" ></text>
</g>
<g >
<title>sync.(*Pool).Put (36 samples, 0.01%)</title><rect x="67.5" y="677" width="0.1" height="15.0" fill="rgb(215,89,13)" rx="2" ry="2" />
<text  x="70.52" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacketBuffer (409 samples, 0.13%)</title><rect x="1027.6" y="661" width="1.5" height="15.0" fill="rgb(253,61,25)" rx="2" ry="2" />
<text  x="1030.60" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).CooperativelySchedulesAddressSpace (77 samples, 0.02%)</title><rect x="83.2" y="693" width="0.3" height="15.0" fill="rgb(224,24,31)" rx="2" ry="2" />
<text  x="86.21" y="703.5" ></text>
</g>
<g >
<title>runtime.usleep (43 samples, 0.01%)</title><rect x="29.2" y="837" width="0.2" height="15.0" fill="rgb(235,105,37)" rx="2" ry="2" />
<text  x="32.21" y="847.5" ></text>
</g>
<g >
<title>runtime.systemstack (29 samples, 0.01%)</title><rect x="50.9" y="581" width="0.1" height="15.0" fill="rgb(234,4,5)" rx="2" ry="2" />
<text  x="53.87" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="1083.3" y="373" width="0.2" height="15.0" fill="rgb(210,35,26)" rx="2" ry="2" />
<text  x="1086.27" y="383.5" ></text>
</g>
<g >
<title>runtime.step (28 samples, 0.01%)</title><rect x="1055.5" y="725" width="0.1" height="15.0" fill="rgb(245,215,24)" rx="2" ry="2" />
<text  x="1058.49" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).EventRegister (33 samples, 0.01%)</title><rect x="71.8" y="709" width="0.1" height="15.0" fill="rgb(216,100,41)" rx="2" ry="2" />
<text  x="74.77" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (267 samples, 0.08%)</title><rect x="1188.7" y="645" width="1.0" height="15.0" fill="rgb(224,26,24)" rx="2" ry="2" />
<text  x="1191.74" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,423 samples, 2.27%)</title><rect x="668.8" y="693" width="26.8" height="15.0" fill="rgb(237,96,46)" rx="2" ry="2" />
<text  x="671.76" y="703.5" >[..</text>
</g>
<g >
<title>nf_ct_get_tuple (69 samples, 0.02%)</title><rect x="961.9" y="453" width="0.2" height="15.0" fill="rgb(209,92,52)" rx="2" ry="2" />
<text  x="964.89" y="463.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (66 samples, 0.02%)</title><rect x="928.8" y="773" width="0.3" height="15.0" fill="rgb(239,135,27)" rx="2" ry="2" />
<text  x="931.84" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,435 samples, 2.28%)</title><rect x="967.7" y="389" width="26.9" height="15.0" fill="rgb(207,122,17)" rx="2" ry="2" />
<text  x="970.69" y="399.5" >[..</text>
</g>
<g >
<title>nf_ct_seq_offset (82 samples, 0.03%)</title><rect x="964.1" y="437" width="0.3" height="15.0" fill="rgb(225,222,15)" rx="2" ry="2" />
<text  x="967.13" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).controlMessages (49 samples, 0.02%)</title><rect x="175.3" y="709" width="0.2" height="15.0" fill="rgb(240,117,11)" rx="2" ry="2" />
<text  x="178.28" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacket (232 samples, 0.07%)</title><rect x="1091.2" y="661" width="0.8" height="15.0" fill="rgb(217,181,5)" rx="2" ry="2" />
<text  x="1094.19" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (379 samples, 0.12%)</title><rect x="920.5" y="629" width="1.4" height="15.0" fill="rgb(252,39,28)" rx="2" ry="2" />
<text  x="923.50" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.AddressWithPrefix.Subnet (47 samples, 0.01%)</title><rect x="724.3" y="725" width="0.2" height="15.0" fill="rgb(216,102,25)" rx="2" ry="2" />
<text  x="727.30" y="735.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (40 samples, 0.01%)</title><rect x="1106.5" y="821" width="0.1" height="15.0" fill="rgb(240,172,34)" rx="2" ry="2" />
<text  x="1109.47" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (143 samples, 0.04%)</title><rect x="1060.3" y="645" width="0.5" height="15.0" fill="rgb(243,10,27)" rx="2" ry="2" />
<text  x="1063.29" y="655.5" ></text>
</g>
<g >
<title>runtime.assertI2I2 (41 samples, 0.01%)</title><rect x="1035.5" y="725" width="0.1" height="15.0" fill="rgb(251,67,9)" rx="2" ry="2" />
<text  x="1038.48" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).Readiness (338 samples, 0.10%)</title><rect x="165.5" y="725" width="1.2" height="15.0" fill="rgb(244,4,11)" rx="2" ry="2" />
<text  x="168.49" y="735.5" ></text>
</g>
<g >
<title>runtime.mapaccess1_fast64 (528 samples, 0.16%)</title><rect x="454.3" y="805" width="1.9" height="15.0" fill="rgb(237,160,53)" rx="2" ry="2" />
<text  x="457.32" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,573 samples, 0.48%)</title><rect x="752.4" y="421" width="5.7" height="15.0" fill="rgb(239,126,7)" rx="2" ry="2" />
<text  x="755.38" y="431.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).RecvMsg (1,123 samples, 0.34%)</title><rect x="174.7" y="741" width="4.1" height="15.0" fill="rgb(211,74,47)" rx="2" ry="2" />
<text  x="177.69" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (88 samples, 0.03%)</title><rect x="23.0" y="725" width="0.3" height="15.0" fill="rgb(244,210,38)" rx="2" ry="2" />
<text  x="25.95" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (50 samples, 0.02%)</title><rect x="1081.4" y="597" width="0.1" height="15.0" fill="rgb(216,215,46)" rx="2" ry="2" />
<text  x="1084.36" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,368 samples, 1.03%)</title><rect x="136.2" y="549" width="12.2" height="15.0" fill="rgb(220,217,26)" rx="2" ry="2" />
<text  x="139.24" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (46 samples, 0.01%)</title><rect x="963.9" y="437" width="0.1" height="15.0" fill="rgb(254,105,14)" rx="2" ry="2" />
<text  x="966.87" y="447.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (6,166 samples, 1.89%)</title><rect x="970.3" y="293" width="22.3" height="15.0" fill="rgb(219,152,29)" rx="2" ry="2" />
<text  x="973.30" y="303.5" >[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendTCP (1,292 samples, 0.40%)</title><rect x="1089.8" y="789" width="4.7" height="15.0" fill="rgb(227,75,17)" rx="2" ry="2" />
<text  x="1092.84" y="799.5" ></text>
</g>
<g >
<title>runtime.newproc.func1 (224 samples, 0.07%)</title><rect x="1005.3" y="821" width="0.8" height="15.0" fill="rgb(229,96,25)" rx="2" ry="2" />
<text  x="1008.28" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (30 samples, 0.01%)</title><rect x="1007.5" y="533" width="0.2" height="15.0" fill="rgb(215,169,9)" rx="2" ry="2" />
<text  x="1010.55" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*handshake).handleSegment (3,050 samples, 0.93%)</title><rect x="1008.2" y="821" width="11.0" height="15.0" fill="rgb(225,157,29)" rx="2" ry="2" />
<text  x="1011.16" y="831.5" ></text>
</g>
<g >
<title>runtime.stackcacherefill (77 samples, 0.02%)</title><rect x="1018.0" y="725" width="0.2" height="15.0" fill="rgb(216,175,15)" rx="2" ry="2" />
<text  x="1020.97" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (67 samples, 0.02%)</title><rect x="163.2" y="725" width="0.3" height="15.0" fill="rgb(247,184,4)" rx="2" ry="2" />
<text  x="166.22" y="735.5" ></text>
</g>
<g >
<title>runtime.lock2 (162 samples, 0.05%)</title><rect x="115.9" y="629" width="0.6" height="15.0" fill="rgb(254,63,30)" rx="2" ry="2" />
<text  x="118.88" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (164 samples, 0.05%)</title><rect x="1119.1" y="709" width="0.6" height="15.0" fill="rgb(229,202,28)" rx="2" ry="2" />
<text  x="1122.13" y="719.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (50,042 samples, 15.34%)</title><rect x="457.0" y="805" width="181.0" height="15.0" fill="rgb(214,136,4)" rx="2" ry="2" />
<text  x="460.05" y="815.5" >syscall.RawSyscall6</text>
</g>
<g >
<title>runtime.dodeltimer0 (38 samples, 0.01%)</title><rect x="1069.1" y="773" width="0.2" height="15.0" fill="rgb(226,146,2)" rx="2" ry="2" />
<text  x="1072.12" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).DeliverNetworkPacket (100 samples, 0.03%)</title><rect x="796.6" y="805" width="0.4" height="15.0" fill="rgb(227,65,21)" rx="2" ry="2" />
<text  x="799.65" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (95 samples, 0.03%)</title><rect x="1120.0" y="789" width="0.4" height="15.0" fill="rgb(251,217,34)" rx="2" ry="2" />
<text  x="1123.04" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverNetworkPacket (92 samples, 0.03%)</title><rect x="19.7" y="869" width="0.3" height="15.0" fill="rgb(206,109,29)" rx="2" ry="2" />
<text  x="22.66" y="879.5" ></text>
</g>
<g >
<title>runtime.siftdownTimer (52 samples, 0.02%)</title><rect x="113.9" y="565" width="0.2" height="15.0" fill="rgb(238,64,51)" rx="2" ry="2" />
<text  x="116.87" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*pmaSet).Find (141 samples, 0.04%)</title><rect x="75.6" y="677" width="0.5" height="15.0" fill="rgb(207,82,46)" rx="2" ry="2" />
<text  x="78.63" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (36 samples, 0.01%)</title><rect x="1079.2" y="645" width="0.2" height="15.0" fill="rgb(212,61,26)" rx="2" ry="2" />
<text  x="1082.24" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MaxHeaderLength (42 samples, 0.01%)</title><rect x="180.6" y="565" width="0.2" height="15.0" fill="rgb(212,137,5)" rx="2" ry="2" />
<text  x="183.65" y="575.5" ></text>
</g>
<g >
<title>runtime.startm (107 samples, 0.03%)</title><rect x="23.3" y="789" width="0.4" height="15.0" fill="rgb(242,113,54)" rx="2" ry="2" />
<text  x="26.32" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*socketOpsCommon).nonBlockingRead (949 samples, 0.29%)</title><rect x="175.0" y="725" width="3.5" height="15.0" fill="rgb(245,170,42)" rx="2" ry="2" />
<text  x="178.04" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (195 samples, 0.06%)</title><rect x="1064.1" y="549" width="0.7" height="15.0" fill="rgb(214,136,17)" rx="2" ry="2" />
<text  x="1067.09" y="559.5" ></text>
</g>
<g >
<title>runtime.entersyscall (500 samples, 0.15%)</title><rect x="446.7" y="757" width="1.8" height="15.0" fill="rgb(219,88,20)" rx="2" ry="2" />
<text  x="449.69" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.writev (3,636 samples, 1.11%)</title><rect x="190.2" y="757" width="13.2" height="15.0" fill="rgb(252,62,44)" rx="2" ry="2" />
<text  x="193.25" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (39 samples, 0.01%)</title><rect x="52.0" y="629" width="0.2" height="15.0" fill="rgb(208,61,19)" rx="2" ry="2" />
<text  x="55.02" y="639.5" ></text>
</g>
<g >
<title>runtime.findrunnable (1,865 samples, 0.57%)</title><rect x="1058.3" y="757" width="6.7" height="15.0" fill="rgb(232,50,41)" rx="2" ry="2" />
<text  x="1061.27" y="767.5" ></text>
</g>
<g >
<title>runtime.funcspdelta (166 samples, 0.05%)</title><rect x="1097.2" y="725" width="0.6" height="15.0" fill="rgb(240,74,2)" rx="2" ry="2" />
<text  x="1100.20" y="735.5" ></text>
</g>
<g >
<title>time.(*Timer).Reset (57 samples, 0.02%)</title><rect x="216.0" y="645" width="0.2" height="15.0" fill="rgb(230,157,21)" rx="2" ry="2" />
<text  x="219.02" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.EpollCtl (911 samples, 0.28%)</title><rect x="69.6" y="773" width="3.3" height="15.0" fill="rgb(224,7,52)" rx="2" ry="2" />
<text  x="72.58" y="783.5" ></text>
</g>
<g >
<title>time.AfterFunc (100 samples, 0.03%)</title><rect x="1069.0" y="837" width="0.3" height="15.0" fill="rgb(250,35,42)" rx="2" ry="2" />
<text  x="1071.96" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (61 samples, 0.02%)</title><rect x="1189.8" y="597" width="0.2" height="15.0" fill="rgb(228,15,49)" rx="2" ry="2" />
<text  x="1192.77" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="18.1" y="629" width="0.1" height="15.0" fill="rgb(242,132,4)" rx="2" ry="2" />
<text  x="21.06" y="639.5" ></text>
</g>
<g >
<title>runtime.pcvalue (68 samples, 0.02%)</title><rect x="1184.4" y="805" width="0.2" height="15.0" fill="rgb(249,120,14)" rx="2" ry="2" />
<text  x="1187.38" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (153 samples, 0.05%)</title><rect x="451.7" y="565" width="0.5" height="15.0" fill="rgb(237,16,13)" rx="2" ry="2" />
<text  x="454.68" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Get (173 samples, 0.05%)</title><rect x="78.5" y="741" width="0.6" height="15.0" fill="rgb(210,184,4)" rx="2" ry="2" />
<text  x="81.50" y="751.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (34 samples, 0.01%)</title><rect x="1094.3" y="709" width="0.1" height="15.0" fill="rgb(207,87,46)" rx="2" ry="2" />
<text  x="1097.28" y="719.5" ></text>
</g>
<g >
<title>runtime.newobject (36 samples, 0.01%)</title><rect x="1048.6" y="677" width="0.1" height="15.0" fill="rgb(224,18,21)" rx="2" ry="2" />
<text  x="1051.56" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (63 samples, 0.02%)</title><rect x="1036.8" y="677" width="0.2" height="15.0" fill="rgb(208,66,13)" rx="2" ry="2" />
<text  x="1039.82" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.ClockGettime (77 samples, 0.02%)</title><rect x="219.0" y="789" width="0.3" height="15.0" fill="rgb(220,6,7)" rx="2" ry="2" />
<text  x="222.03" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).lockAppendMu (60 samples, 0.02%)</title><rect x="191.5" y="725" width="0.3" height="15.0" fill="rgb(214,142,11)" rx="2" ry="2" />
<text  x="194.55" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).WritePacket (632 samples, 0.19%)</title><rect x="1073.2" y="693" width="2.3" height="15.0" fill="rgb(209,118,21)" rx="2" ry="2" />
<text  x="1076.22" y="703.5" ></text>
</g>
<g >
<title>runtime.futex (881 samples, 0.27%)</title><rect x="1123.6" y="773" width="3.2" height="15.0" fill="rgb(209,10,24)" rx="2" ry="2" />
<text  x="1126.61" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (47 samples, 0.01%)</title><rect x="1119.6" y="629" width="0.1" height="15.0" fill="rgb(213,208,51)" rx="2" ry="2" />
<text  x="1122.56" y="639.5" ></text>
</g>
<g >
<title>runtime.newobject (128 samples, 0.04%)</title><rect x="51.3" y="677" width="0.5" height="15.0" fill="rgb(210,82,31)" rx="2" ry="2" />
<text  x="54.35" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).AcquireAssignedAddressOrMatching (105 samples, 0.03%)</title><rect x="1034.8" y="709" width="0.4" height="15.0" fill="rgb(234,139,11)" rx="2" ry="2" />
<text  x="1037.80" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/rawfile.BlockingPoll (32 samples, 0.01%)</title><rect x="824.4" y="725" width="0.1" height="15.0" fill="rgb(216,108,27)" rx="2" ry="2" />
<text  x="827.38" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (223 samples, 0.07%)</title><rect x="1028.1" y="581" width="0.8" height="15.0" fill="rgb(220,32,14)" rx="2" ry="2" />
<text  x="1031.06" y="591.5" ></text>
</g>
<g >
<title>ext4_file_write_iter (902 samples, 0.28%)</title><rect x="197.9" y="357" width="3.3" height="15.0" fill="rgb(243,97,2)" rx="2" ry="2" />
<text  x="200.93" y="367.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/ports.(*PortManager).ReserveTuple (289 samples, 0.09%)</title><rect x="1025.3" y="789" width="1.0" height="15.0" fill="rgb(251,141,5)" rx="2" ry="2" />
<text  x="1028.28" y="799.5" ></text>
</g>
<g >
<title>runtime.goexit0 (88 samples, 0.03%)</title><rect x="17.2" y="837" width="0.3" height="15.0" fill="rgb(248,209,51)" rx="2" ry="2" />
<text  x="20.18" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*listenContext).cleanupCompletedHandshake (177 samples, 0.05%)</title><rect x="1020.4" y="853" width="0.7" height="15.0" fill="rgb(214,212,18)" rx="2" ry="2" />
<text  x="1023.44" y="863.5" ></text>
</g>
<g >
<title>runtime.getempty (46 samples, 0.01%)</title><rect x="1143.7" y="693" width="0.1" height="15.0" fill="rgb(211,17,35)" rx="2" ry="2" />
<text  x="1146.67" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (47 samples, 0.01%)</title><rect x="776.5" y="661" width="0.1" height="15.0" fill="rgb(249,217,23)" rx="2" ry="2" />
<text  x="779.47" y="671.5" ></text>
</g>
<g >
<title>runtime.runOneTimer (30 samples, 0.01%)</title><rect x="1108.0" y="741" width="0.1" height="15.0" fill="rgb(227,79,31)" rx="2" ry="2" />
<text  x="1110.98" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (31 samples, 0.01%)</title><rect x="210.2" y="405" width="0.1" height="15.0" fill="rgb(246,166,29)" rx="2" ry="2" />
<text  x="213.23" y="415.5" ></text>
</g>
<g >
<title>runtime.addtimer (73 samples, 0.02%)</title><rect x="1011.1" y="709" width="0.3" height="15.0" fill="rgb(250,152,5)" rx="2" ry="2" />
<text  x="1014.12" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*inodeOperations).UnstableAttr (89 samples, 0.03%)</title><rect x="188.3" y="597" width="0.3" height="15.0" fill="rgb(242,74,13)" rx="2" ry="2" />
<text  x="191.31" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (275 samples, 0.08%)</title><rect x="703.9" y="709" width="1.0" height="15.0" fill="rgb(250,56,51)" rx="2" ry="2" />
<text  x="706.88" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (51 samples, 0.02%)</title><rect x="699.9" y="757" width="0.2" height="15.0" fill="rgb(207,32,14)" rx="2" ry="2" />
<text  x="702.90" y="767.5" ></text>
</g>
<g >
<title>runtime.notesleep (401 samples, 0.12%)</title><rect x="1186.2" y="789" width="1.4" height="15.0" fill="rgb(221,75,30)" rx="2" ry="2" />
<text  x="1189.18" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/socket.NewDirent (1,290 samples, 0.40%)</title><rect x="48.3" y="709" width="4.7" height="15.0" fill="rgb(234,196,32)" rx="2" ry="2" />
<text  x="51.32" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (40 samples, 0.01%)</title><rect x="18.0" y="677" width="0.2" height="15.0" fill="rgb(219,127,22)" rx="2" ry="2" />
<text  x="21.04" y="687.5" ></text>
</g>
<g >
<title>runtime.step (193 samples, 0.06%)</title><rect x="1136.6" y="725" width="0.7" height="15.0" fill="rgb(226,3,48)" rx="2" ry="2" />
<text  x="1139.63" y="735.5" ></text>
</g>
<g >
<title>fmt.Sprintf (286 samples, 0.09%)</title><rect x="48.5" y="693" width="1.0" height="15.0" fill="rgb(218,219,29)" rx="2" ry="2" />
<text  x="51.49" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindNICNameFromID (189 samples, 0.06%)</title><rect x="779.2" y="725" width="0.7" height="15.0" fill="rgb(227,17,10)" rx="2" ry="2" />
<text  x="782.17" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).MainAddress (585 samples, 0.18%)</title><rect x="725.0" y="709" width="2.1" height="15.0" fill="rgb(217,60,19)" rx="2" ry="2" />
<text  x="727.99" y="719.5" ></text>
</g>
<g >
<title>runtime.mallocgc (307 samples, 0.09%)</title><rect x="928.1" y="789" width="1.2" height="15.0" fill="rgb(249,32,0)" rx="2" ry="2" />
<text  x="931.15" y="799.5" ></text>
</g>
<g >
<title>runtime.systemstack (46 samples, 0.01%)</title><rect x="1085.3" y="661" width="0.2" height="15.0" fill="rgb(238,68,12)" rx="2" ry="2" />
<text  x="1088.30" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (59,442 samples, 18.22%)</title><rect x="231.7" y="757" width="215.0" height="15.0" fill="rgb(250,42,20)" rx="2" ry="2" />
<text  x="234.69" y="767.5" >[[kernel.kallsyms]]</text>
</g>
<g >
<title>runtime.gfput (31 samples, 0.01%)</title><rect x="1112.2" y="837" width="0.1" height="15.0" fill="rgb(225,205,9)" rx="2" ry="2" />
<text  x="1115.21" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).addIPHeader (59 samples, 0.02%)</title><rect x="209.5" y="549" width="0.2" height="15.0" fill="rgb(208,9,5)" rx="2" ry="2" />
<text  x="212.46" y="559.5" ></text>
</g>
<g >
<title>runtime.newobject (58 samples, 0.02%)</title><rect x="1075.0" y="613" width="0.2" height="15.0" fill="rgb(238,99,45)" rx="2" ry="2" />
<text  x="1078.04" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (78 samples, 0.02%)</title><rect x="23.0" y="677" width="0.3" height="15.0" fill="rgb(236,83,6)" rx="2" ry="2" />
<text  x="25.99" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).resolvedFields (246 samples, 0.08%)</title><rect x="1029.1" y="661" width="0.9" height="15.0" fill="rgb(234,21,10)" rx="2" ry="2" />
<text  x="1032.07" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*MountNamespace).FindLink (1,243 samples, 0.38%)</title><rect x="185.1" y="725" width="4.5" height="15.0" fill="rgb(242,110,24)" rx="2" ry="2" />
<text  x="188.11" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isValidForOutgoingRLocked (75 samples, 0.02%)</title><rect x="1093.1" y="725" width="0.3" height="15.0" fill="rgb(213,189,32)" rx="2" ry="2" />
<text  x="1096.11" y="735.5" ></text>
</g>
<g >
<title>runtime.mallocgc (45 samples, 0.01%)</title><rect x="48.0" y="677" width="0.2" height="15.0" fill="rgb(222,103,50)" rx="2" ry="2" />
<text  x="51.02" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="990.7" y="53" width="0.1" height="15.0" fill="rgb(209,156,32)" rx="2" ry="2" />
<text  x="993.68" y="63.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (136 samples, 0.04%)</title><rect x="12.9" y="709" width="0.5" height="15.0" fill="rgb(243,142,45)" rx="2" ry="2" />
<text  x="15.90" y="719.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (579 samples, 0.18%)</title><rect x="765.2" y="581" width="2.1" height="15.0" fill="rgb(216,112,14)" rx="2" ry="2" />
<text  x="768.25" y="591.5" ></text>
</g>
<g >
<title>runtime.(*mheap).alloc.func1 (44 samples, 0.01%)</title><rect x="711.5" y="693" width="0.1" height="15.0" fill="rgb(215,144,38)" rx="2" ry="2" />
<text  x="714.46" y="703.5" ></text>
</g>
<g >
<title>runtime.mallocgc (31 samples, 0.01%)</title><rect x="52.5" y="677" width="0.1" height="15.0" fill="rgb(221,73,46)" rx="2" ry="2" />
<text  x="55.47" y="687.5" ></text>
</g>
<g >
<title>runtime.startm (1,775 samples, 0.54%)</title><rect x="149.9" y="613" width="6.5" height="15.0" fill="rgb(220,73,54)" rx="2" ry="2" />
<text  x="152.95" y="623.5" ></text>
</g>
<g >
<title>runtime.gcBgMarkWorker.func2 (14,528 samples, 4.45%)</title><rect x="1131.0" y="853" width="52.5" height="15.0" fill="rgb(233,89,24)" rx="2" ry="2" />
<text  x="1133.96" y="863.5" >runti..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (103 samples, 0.03%)</title><rect x="1066.2" y="533" width="0.3" height="15.0" fill="rgb(245,75,3)" rx="2" ry="2" />
<text  x="1069.15" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).Fetch (7,202 samples, 2.21%)</title><rect x="896.7" y="837" width="26.0" height="15.0" fill="rgb(247,75,52)" rx="2" ry="2" />
<text  x="899.68" y="847.5" >g..</text>
</g>
<g >
<title>nf_ct_deliver_cached_events (270 samples, 0.08%)</title><rect x="992.7" y="293" width="1.0" height="15.0" fill="rgb(240,226,3)" rx="2" ry="2" />
<text  x="995.73" y="303.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*backoffTimer).stop (32 samples, 0.01%)</title><rect x="1007.9" y="837" width="0.1" height="15.0" fill="rgb(232,219,8)" rx="2" ry="2" />
<text  x="1010.88" y="847.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (84 samples, 0.03%)</title><rect x="50.7" y="597" width="0.3" height="15.0" fill="rgb(243,124,20)" rx="2" ry="2" />
<text  x="53.67" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.Rdtsc (34 samples, 0.01%)</title><rect x="60.3" y="677" width="0.1" height="15.0" fill="rgb(209,120,16)" rx="2" ry="2" />
<text  x="63.31" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (66 samples, 0.02%)</title><rect x="1091.3" y="613" width="0.3" height="15.0" fill="rgb(244,65,40)" rx="2" ry="2" />
<text  x="1094.32" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).DecRef.func1 (889 samples, 0.27%)</title><rect x="63.8" y="725" width="3.2" height="15.0" fill="rgb(208,201,54)" rx="2" ry="2" />
<text  x="66.80" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Kernel).incRunningTasks (191 samples, 0.06%)</title><rect x="82.2" y="677" width="0.6" height="15.0" fill="rgb(231,132,44)" rx="2" ry="2" />
<text  x="85.15" y="687.5" ></text>
</g>
<g >
<title>runtime.park_m (165 samples, 0.05%)</title><rect x="451.6" y="725" width="0.6" height="15.0" fill="rgb(226,29,30)" rx="2" ry="2" />
<text  x="454.64" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.FromIOWriter.WriteFromBlocks (47 samples, 0.01%)</title><rect x="21.6" y="629" width="0.1" height="15.0" fill="rgb(242,12,42)" rx="2" ry="2" />
<text  x="24.57" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).makeOptions (86 samples, 0.03%)</title><rect x="1010.0" y="741" width="0.3" height="15.0" fill="rgb(224,33,53)" rx="2" ry="2" />
<text  x="1013.03" y="751.5" ></text>
</g>
<g >
<title>runtime.runtimer (77 samples, 0.02%)</title><rect x="1118.4" y="789" width="0.3" height="15.0" fill="rgb(213,109,26)" rx="2" ry="2" />
<text  x="1121.42" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).WritePacket (863 samples, 0.26%)</title><rect x="1027.2" y="725" width="3.1" height="15.0" fill="rgb(224,98,27)" rx="2" ry="2" />
<text  x="1030.16" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.(*IPv4).DestinationAddress (138 samples, 0.04%)</title><rect x="762.0" y="613" width="0.5" height="15.0" fill="rgb(249,141,38)" rx="2" ry="2" />
<text  x="764.99" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.DeleteDanglingEndpoint (52 samples, 0.02%)</title><rect x="1099.9" y="821" width="0.2" height="15.0" fill="rgb(219,11,7)" rx="2" ry="2" />
<text  x="1102.90" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (33 samples, 0.01%)</title><rect x="1051.2" y="581" width="0.2" height="15.0" fill="rgb(235,203,27)" rx="2" ry="2" />
<text  x="1054.24" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).EventRegister (39 samples, 0.01%)</title><rect x="170.2" y="741" width="0.1" height="15.0" fill="rgb(235,33,19)" rx="2" ry="2" />
<text  x="173.21" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (37 samples, 0.01%)</title><rect x="23.6" y="645" width="0.1" height="15.0" fill="rgb(205,166,35)" rx="2" ry="2" />
<text  x="26.57" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isValidForOutgoing (83 samples, 0.03%)</title><rect x="1075.5" y="693" width="0.3" height="15.0" fill="rgb(216,23,34)" rx="2" ry="2" />
<text  x="1078.53" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,308 samples, 2.24%)</title><rect x="968.1" y="373" width="26.5" height="15.0" fill="rgb(229,37,19)" rx="2" ry="2" />
<text  x="971.15" y="383.5" >[..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="452.3" y="565" width="0.2" height="15.0" fill="rgb(247,18,23)" rx="2" ry="2" />
<text  x="455.31" y="575.5" ></text>
</g>
<g >
<title>runtime.(*spanSet).pop (48 samples, 0.01%)</title><rect x="1109.9" y="821" width="0.2" height="15.0" fill="rgb(231,186,23)" rx="2" ry="2" />
<text  x="1112.90" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (111 samples, 0.03%)</title><rect x="230.7" y="757" width="0.4" height="15.0" fill="rgb(252,207,20)" rx="2" ry="2" />
<text  x="233.66" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).DecRef (107 samples, 0.03%)</title><rect x="64.0" y="645" width="0.4" height="15.0" fill="rgb(220,7,40)" rx="2" ry="2" />
<text  x="66.97" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).lockAppendMu (37 samples, 0.01%)</title><rect x="205.0" y="725" width="0.1" height="15.0" fill="rgb(212,48,9)" rx="2" ry="2" />
<text  x="208.00" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*recvMMsgDispatcher).dispatch (51,870 samples, 15.90%)</title><rect x="707.3" y="837" width="187.6" height="15.0" fill="rgb(222,209,8)" rx="2" ry="2" />
<text  x="710.31" y="847.5" >gvisor.dev/gvisor/pkg/tc..</text>
</g>
<g >
<title>runtime.funcMaxSPDelta (48 samples, 0.01%)</title><rect x="1018.5" y="757" width="0.1" height="15.0" fill="rgb(233,216,7)" rx="2" ry="2" />
<text  x="1021.46" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*WeakRef).Drop (96 samples, 0.03%)</title><rect x="67.3" y="693" width="0.4" height="15.0" fill="rgb(245,210,15)" rx="2" ry="2" />
<text  x="70.32" y="703.5" ></text>
</g>
<g >
<title>runtime.isSystemGoroutine (244 samples, 0.07%)</title><rect x="1112.3" y="837" width="0.9" height="15.0" fill="rgb(212,79,10)" rx="2" ry="2" />
<text  x="1115.32" y="847.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (28 samples, 0.01%)</title><rect x="1035.2" y="677" width="0.1" height="15.0" fill="rgb(209,125,28)" rx="2" ry="2" />
<text  x="1038.25" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).protocolMainLoop.func1 (104 samples, 0.03%)</title><rect x="1069.8" y="853" width="0.3" height="15.0" fill="rgb(245,55,42)" rx="2" ry="2" />
<text  x="1072.75" y="863.5" ></text>
</g>
<g >
<title>runtime.findrunnable (835 samples, 0.26%)</title><rect x="699.1" y="789" width="3.0" height="15.0" fill="rgb(254,170,30)" rx="2" ry="2" />
<text  x="702.05" y="799.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast64 (41 samples, 0.01%)</title><rect x="773.6" y="677" width="0.2" height="15.0" fill="rgb(215,179,16)" rx="2" ry="2" />
<text  x="776.62" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (74 samples, 0.02%)</title><rect x="452.2" y="757" width="0.3" height="15.0" fill="rgb(224,3,47)" rx="2" ry="2" />
<text  x="455.24" y="767.5" ></text>
</g>
<g >
<title>runtime.runqget (30 samples, 0.01%)</title><rect x="922.0" y="757" width="0.2" height="15.0" fill="rgb(215,134,52)" rx="2" ry="2" />
<text  x="925.04" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (110 samples, 0.03%)</title><rect x="732.8" y="677" width="0.4" height="15.0" fill="rgb(206,65,42)" rx="2" ry="2" />
<text  x="735.85" y="687.5" ></text>
</g>
<g >
<title>runtime.newobject (190 samples, 0.06%)</title><rect x="177.8" y="709" width="0.7" height="15.0" fill="rgb(218,161,34)" rx="2" ry="2" />
<text  x="180.77" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (68 samples, 0.02%)</title><rect x="1150.8" y="677" width="0.2" height="15.0" fill="rgb(235,126,19)" rx="2" ry="2" />
<text  x="1153.78" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).decAddressRef (216 samples, 0.07%)</title><rect x="776.2" y="693" width="0.8" height="15.0" fill="rgb(241,126,15)" rx="2" ry="2" />
<text  x="779.23" y="703.5" ></text>
</g>
<g >
<title>sync.(*Pool).getSlow (53 samples, 0.02%)</title><rect x="45.8" y="677" width="0.2" height="15.0" fill="rgb(220,168,23)" rx="2" ry="2" />
<text  x="48.83" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (389 samples, 0.12%)</title><rect x="705.3" y="565" width="1.4" height="15.0" fill="rgb(233,98,10)" rx="2" ry="2" />
<text  x="708.29" y="575.5" ></text>
</g>
<g >
<title>runtime.stackalloc (789 samples, 0.24%)</title><rect x="1150.7" y="741" width="2.8" height="15.0" fill="rgb(208,104,42)" rx="2" ry="2" />
<text  x="1153.66" y="751.5" ></text>
</g>
<g >
<title>runtime.futex (536 samples, 0.16%)</title><rect x="1187.8" y="757" width="1.9" height="15.0" fill="rgb(235,16,37)" rx="2" ry="2" />
<text  x="1190.77" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (125 samples, 0.04%)</title><rect x="1060.4" y="629" width="0.4" height="15.0" fill="rgb(224,104,13)" rx="2" ry="2" />
<text  x="1063.35" y="639.5" ></text>
</g>
<g >
<title>runtime.mstart1 (47 samples, 0.01%)</title><rect x="16.8" y="741" width="0.1" height="15.0" fill="rgb(228,38,5)" rx="2" ry="2" />
<text  x="19.75" y="751.5" ></text>
</g>
<g >
<title>runtime.futex (1,698 samples, 0.52%)</title><rect x="150.2" y="581" width="6.1" height="15.0" fill="rgb(231,59,35)" rx="2" ry="2" />
<text  x="153.15" y="591.5" ></text>
</g>
<g >
<title>runtime.wakeNetPoller (247 samples, 0.08%)</title><rect x="1082.6" y="597" width="0.9" height="15.0" fill="rgb(240,184,12)" rx="2" ry="2" />
<text  x="1085.57" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Accept4 (4,327 samples, 1.33%)</title><rect x="43.3" y="773" width="15.6" height="15.0" fill="rgb(210,184,43)" rx="2" ry="2" />
<text  x="46.28" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).newPassiveHandshake (239 samples, 0.07%)</title><rect x="1024.4" y="805" width="0.9" height="15.0" fill="rgb(215,119,34)" rx="2" ry="2" />
<text  x="1027.39" y="815.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast32 (50 samples, 0.02%)</title><rect x="779.7" y="709" width="0.2" height="15.0" fill="rgb(249,170,26)" rx="2" ry="2" />
<text  x="782.67" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).GSOMaxSize (68 samples, 0.02%)</title><rect x="1038.2" y="741" width="0.3" height="15.0" fill="rgb(239,128,28)" rx="2" ry="2" />
<text  x="1041.22" y="751.5" ></text>
</g>
<g >
<title>runtime.(*mcache).prepareForSweep (100 samples, 0.03%)</title><rect x="843.9" y="725" width="0.4" height="15.0" fill="rgb(226,97,37)" rx="2" ry="2" />
<text  x="846.92" y="735.5" ></text>
</g>
<g >
<title>runtime.funcMaxSPDelta (44 samples, 0.01%)</title><rect x="1098.9" y="757" width="0.1" height="15.0" fill="rgb(243,177,20)" rx="2" ry="2" />
<text  x="1101.89" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).Destroy (193 samples, 0.06%)</title><rect x="1084.9" y="693" width="0.7" height="15.0" fill="rgb(209,161,6)" rx="2" ry="2" />
<text  x="1087.94" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (33 samples, 0.01%)</title><rect x="13.9" y="565" width="0.1" height="15.0" fill="rgb(225,11,54)" rx="2" ry="2" />
<text  x="16.86" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (243 samples, 0.07%)</title><rect x="212.2" y="165" width="0.9" height="15.0" fill="rgb(227,187,38)" rx="2" ry="2" />
<text  x="215.24" y="175.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doSyscallInvoke (37 samples, 0.01%)</title><rect x="220.8" y="821" width="0.2" height="15.0" fill="rgb(238,109,19)" rx="2" ry="2" />
<text  x="223.84" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.fileOpOn (1,370 samples, 0.42%)</title><rect x="184.9" y="757" width="5.0" height="15.0" fill="rgb(222,16,0)" rx="2" ry="2" />
<text  x="187.92" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).accountTaskGoroutineEnter (170 samples, 0.05%)</title><rect x="36.7" y="837" width="0.6" height="15.0" fill="rgb(215,102,17)" rx="2" ry="2" />
<text  x="39.73" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (123 samples, 0.04%)</title><rect x="18.2" y="725" width="0.5" height="15.0" fill="rgb(210,160,43)" rx="2" ry="2" />
<text  x="21.22" y="735.5" ></text>
</g>
<g >
<title>runtime.putempty (147 samples, 0.05%)</title><rect x="1146.5" y="773" width="0.6" height="15.0" fill="rgb(244,200,46)" rx="2" ry="2" />
<text  x="1149.52" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Job).Schedule (1,562 samples, 0.48%)</title><rect x="1078.8" y="725" width="5.7" height="15.0" fill="rgb(209,222,12)" rx="2" ry="2" />
<text  x="1081.85" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (139 samples, 0.04%)</title><rect x="1108.5" y="725" width="0.5" height="15.0" fill="rgb(223,144,4)" rx="2" ry="2" />
<text  x="1111.53" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (67 samples, 0.02%)</title><rect x="1189.8" y="741" width="0.2" height="15.0" fill="rgb(243,50,27)" rx="2" ry="2" />
<text  x="1192.75" y="751.5" ></text>
</g>
<g >
<title>runtime.futex (4,632 samples, 1.42%)</title><rect x="824.5" y="725" width="16.7" height="15.0" fill="rgb(219,192,38)" rx="2" ry="2" />
<text  x="827.50" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (15,382 samples, 4.71%)</title><rect x="942.5" y="661" width="55.6" height="15.0" fill="rgb(226,219,36)" rx="2" ry="2" />
<text  x="945.50" y="671.5" >[[ker..</text>
</g>
<g >
<title>runtime.wakep (109 samples, 0.03%)</title><rect x="23.3" y="805" width="0.4" height="15.0" fill="rgb(243,84,51)" rx="2" ry="2" />
<text  x="26.31" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (67 samples, 0.02%)</title><rect x="1189.8" y="725" width="0.2" height="15.0" fill="rgb(205,1,46)" rx="2" ry="2" />
<text  x="1192.75" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).get (68 samples, 0.02%)</title><rect x="174.3" y="725" width="0.3" height="15.0" fill="rgb(205,0,0)" rx="2" ry="2" />
<text  x="177.32" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendSegment (782 samples, 0.24%)</title><rect x="180.2" y="645" width="2.8" height="15.0" fill="rgb(238,191,32)" rx="2" ry="2" />
<text  x="183.17" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.writev (52 samples, 0.02%)</title><rect x="21.6" y="821" width="0.2" height="15.0" fill="rgb(237,104,50)" rx="2" ry="2" />
<text  x="24.57" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (157 samples, 0.05%)</title><rect x="1122.6" y="629" width="0.5" height="15.0" fill="rgb(234,199,16)" rx="2" ry="2" />
<text  x="1125.58" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (64 samples, 0.02%)</title><rect x="15.9" y="805" width="0.2" height="15.0" fill="rgb(246,14,15)" rx="2" ry="2" />
<text  x="18.89" y="815.5" ></text>
</g>
<g >
<title>runtime.scanframeworker (2,450 samples, 0.75%)</title><rect x="1137.6" y="741" width="8.9" height="15.0" fill="rgb(237,145,44)" rx="2" ry="2" />
<text  x="1140.65" y="751.5" ></text>
</g>
<g >
<title>runtime.step (86 samples, 0.03%)</title><rect x="1097.5" y="693" width="0.3" height="15.0" fill="rgb(242,144,14)" rx="2" ry="2" />
<text  x="1100.45" y="703.5" ></text>
</g>
<g >
<title>runtime.notewakeup (66 samples, 0.02%)</title><rect x="28.9" y="821" width="0.3" height="15.0" fill="rgb(209,130,9)" rx="2" ry="2" />
<text  x="31.94" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (215 samples, 0.07%)</title><rect x="1060.0" y="725" width="0.8" height="15.0" fill="rgb(217,190,16)" rx="2" ry="2" />
<text  x="1063.03" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).InotifyEvent (63 samples, 0.02%)</title><rect x="190.3" y="741" width="0.3" height="15.0" fill="rgb(206,165,4)" rx="2" ry="2" />
<text  x="193.32" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (559 samples, 0.17%)</title><rect x="1124.8" y="725" width="2.0" height="15.0" fill="rgb(222,86,22)" rx="2" ry="2" />
<text  x="1127.77" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (486 samples, 0.15%)</title><rect x="1125.0" y="677" width="1.8" height="15.0" fill="rgb(219,181,38)" rx="2" ry="2" />
<text  x="1128.04" y="687.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (28 samples, 0.01%)</title><rect x="182.7" y="517" width="0.1" height="15.0" fill="rgb(236,173,48)" rx="2" ry="2" />
<text  x="185.69" y="527.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (119 samples, 0.04%)</title><rect x="1152.7" y="677" width="0.4" height="15.0" fill="rgb(240,55,36)" rx="2" ry="2" />
<text  x="1155.68" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendRaw (2,089 samples, 0.64%)</title><rect x="208.2" y="629" width="7.6" height="15.0" fill="rgb(245,77,23)" rx="2" ry="2" />
<text  x="211.22" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.selectEndpoint (224 samples, 0.07%)</title><rect x="742.0" y="661" width="0.8" height="15.0" fill="rgb(242,70,10)" rx="2" ry="2" />
<text  x="744.97" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).initialReceiveWindow (85 samples, 0.03%)</title><rect x="1038.8" y="789" width="0.3" height="15.0" fill="rgb(249,37,24)" rx="2" ry="2" />
<text  x="1041.81" y="799.5" ></text>
</g>
<g >
<title>runtime.morestack (429 samples, 0.13%)</title><rect x="1183.5" y="885" width="1.6" height="15.0" fill="rgb(214,29,33)" rx="2" ry="2" />
<text  x="1186.51" y="895.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (66 samples, 0.02%)</title><rect x="17.9" y="741" width="0.3" height="15.0" fill="rgb(215,103,6)" rx="2" ry="2" />
<text  x="20.95" y="751.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (45 samples, 0.01%)</title><rect x="183.2" y="645" width="0.1" height="15.0" fill="rgb(254,228,7)" rx="2" ry="2" />
<text  x="186.18" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (113 samples, 0.03%)</title><rect x="904.5" y="645" width="0.4" height="15.0" fill="rgb(214,172,33)" rx="2" ry="2" />
<text  x="907.49" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/rawfile.callEntersyscallblock (5,401 samples, 1.66%)</title><rect x="822.5" y="821" width="19.5" height="15.0" fill="rgb(240,138,22)" rx="2" ry="2" />
<text  x="825.49" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindRoute (52 samples, 0.02%)</title><rect x="1032.3" y="789" width="0.2" height="15.0" fill="rgb(210,178,20)" rx="2" ry="2" />
<text  x="1035.34" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleSegment (5,202 samples, 1.59%)</title><rect x="1070.3" y="821" width="18.8" height="15.0" fill="rgb(218,150,48)" rx="2" ry="2" />
<text  x="1073.32" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*timer).enable (91 samples, 0.03%)</title><rect x="1088.5" y="789" width="0.3" height="15.0" fill="rgb(214,169,44)" rx="2" ry="2" />
<text  x="1091.47" y="799.5" ></text>
</g>
<g >
<title>[unknown] (1,091 samples, 0.33%)</title><rect x="12.7" y="853" width="3.9" height="15.0" fill="rgb(218,221,21)" rx="2" ry="2" />
<text  x="15.70" y="863.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (102 samples, 0.03%)</title><rect x="50.7" y="645" width="0.3" height="15.0" fill="rgb(229,32,24)" rx="2" ry="2" />
<text  x="53.66" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).HandlePacket (15,663 samples, 4.80%)</title><rect x="723.7" y="741" width="56.6" height="15.0" fill="rgb(221,93,21)" rx="2" ry="2" />
<text  x="726.69" y="751.5" >gvisor..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (44 samples, 0.01%)</title><rect x="1061.3" y="677" width="0.2" height="15.0" fill="rgb(248,6,4)" rx="2" ry="2" />
<text  x="1064.34" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).get (141 samples, 0.04%)</title><rect x="44.2" y="725" width="0.5" height="15.0" fill="rgb(254,93,27)" rx="2" ry="2" />
<text  x="47.23" y="735.5" ></text>
</g>
<g >
<title>runtime.execute (47 samples, 0.01%)</title><rect x="898.2" y="757" width="0.2" height="15.0" fill="rgb(231,173,24)" rx="2" ry="2" />
<text  x="901.19" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="115.7" y="437" width="0.2" height="15.0" fill="rgb(253,160,23)" rx="2" ry="2" />
<text  x="118.72" y="447.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (193 samples, 0.06%)</title><rect x="218.0" y="757" width="0.7" height="15.0" fill="rgb(235,168,32)" rx="2" ry="2" />
<text  x="221.00" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (205 samples, 0.06%)</title><rect x="1127.8" y="693" width="0.8" height="15.0" fill="rgb(239,167,40)" rx="2" ry="2" />
<text  x="1130.82" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/marshal/primitive.(*Uint32).CopyOutN (72 samples, 0.02%)</title><rect x="57.2" y="709" width="0.2" height="15.0" fill="rgb(229,161,31)" rx="2" ry="2" />
<text  x="60.15" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (81 samples, 0.02%)</title><rect x="1007.4" y="661" width="0.3" height="15.0" fill="rgb(227,152,0)" rx="2" ry="2" />
<text  x="1010.37" y="671.5" ></text>
</g>
<g >
<title>runtime.newobject (72 samples, 0.02%)</title><rect x="1043.4" y="821" width="0.3" height="15.0" fill="rgb(253,64,39)" rx="2" ry="2" />
<text  x="1046.39" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,709 samples, 1.75%)</title><rect x="972.0" y="277" width="20.6" height="15.0" fill="rgb(251,30,26)" rx="2" ry="2" />
<text  x="974.95" y="287.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (84 samples, 0.03%)</title><rect x="149.0" y="549" width="0.4" height="15.0" fill="rgb(242,56,15)" rx="2" ry="2" />
<text  x="152.05" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (83 samples, 0.03%)</title><rect x="766.2" y="485" width="0.3" height="15.0" fill="rgb(207,160,41)" rx="2" ry="2" />
<text  x="769.19" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.TcpipAfterFunc (749 samples, 0.23%)</title><rect x="1047.5" y="757" width="2.7" height="15.0" fill="rgb(222,194,5)" rx="2" ry="2" />
<text  x="1050.46" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="1098.0" y="693" width="0.2" height="15.0" fill="rgb(211,147,8)" rx="2" ry="2" />
<text  x="1101.04" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (98 samples, 0.03%)</title><rect x="962.5" y="437" width="0.4" height="15.0" fill="rgb(219,118,2)" rx="2" ry="2" />
<text  x="965.53" y="447.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (124 samples, 0.04%)</title><rect x="1119.3" y="677" width="0.4" height="15.0" fill="rgb(212,152,35)" rx="2" ry="2" />
<text  x="1122.28" y="687.5" ></text>
</g>
<g >
<title>runtime.(*mheap).allocSpan (83 samples, 0.03%)</title><rect x="848.1" y="693" width="0.3" height="15.0" fill="rgb(222,65,25)" rx="2" ry="2" />
<text  x="851.13" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (39 samples, 0.01%)</title><rect x="741.8" y="661" width="0.2" height="15.0" fill="rgb(217,175,19)" rx="2" ry="2" />
<text  x="744.83" y="671.5" ></text>
</g>
<g >
<title>fmt.(*pp).fmtInteger (75 samples, 0.02%)</title><rect x="48.8" y="645" width="0.3" height="15.0" fill="rgb(246,87,37)" rx="2" ry="2" />
<text  x="51.83" y="655.5" ></text>
</g>
<g >
<title>br_handle_frame (11,104 samples, 3.40%)</title><rect x="956.5" y="533" width="40.2" height="15.0" fill="rgb(210,108,6)" rx="2" ry="2" />
<text  x="959.54" y="543.5" >br_..</text>
</g>
<g >
<title>runtime.newobject (44 samples, 0.01%)</title><rect x="1049.8" y="725" width="0.1" height="15.0" fill="rgb(235,200,31)" rx="2" ry="2" />
<text  x="1052.77" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (857 samples, 0.26%)</title><rect x="915.6" y="565" width="3.1" height="15.0" fill="rgb(236,228,41)" rx="2" ry="2" />
<text  x="918.63" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MaxHeaderLength (100 samples, 0.03%)</title><rect x="208.9" y="581" width="0.4" height="15.0" fill="rgb(228,208,18)" rx="2" ry="2" />
<text  x="211.90" y="591.5" ></text>
</g>
<g >
<title>runtime.checkTimers (92 samples, 0.03%)</title><rect x="1113.6" y="821" width="0.3" height="15.0" fill="rgb(220,149,41)" rx="2" ry="2" />
<text  x="1116.60" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.NewFile (185 samples, 0.06%)</title><rect x="47.5" y="709" width="0.7" height="15.0" fill="rgb(219,26,23)" rx="2" ry="2" />
<text  x="50.53" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,127 samples, 0.35%)</title><rect x="952.5" y="501" width="4.0" height="15.0" fill="rgb(212,94,52)" rx="2" ry="2" />
<text  x="955.47" y="511.5" ></text>
</g>
<g >
<title>runtime.gentraceback (3,451 samples, 1.06%)</title><rect x="1134.0" y="773" width="12.5" height="15.0" fill="rgb(245,204,35)" rx="2" ry="2" />
<text  x="1137.03" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (926 samples, 0.28%)</title><rect x="915.4" y="581" width="3.3" height="15.0" fill="rgb(214,125,35)" rx="2" ry="2" />
<text  x="918.38" y="591.5" ></text>
</g>
<g >
<title>runtime.funcname (39 samples, 0.01%)</title><rect x="1080.7" y="581" width="0.2" height="15.0" fill="rgb(248,89,25)" rx="2" ry="2" />
<text  x="1083.73" y="591.5" ></text>
</g>
<g >
<title>runtime.getStackMap (368 samples, 0.11%)</title><rect x="1015.2" y="709" width="1.3" height="15.0" fill="rgb(225,153,19)" rx="2" ry="2" />
<text  x="1018.21" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).GetKind (185 samples, 0.06%)</title><rect x="787.7" y="661" width="0.7" height="15.0" fill="rgb(227,221,1)" rx="2" ry="2" />
<text  x="790.69" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (283 samples, 0.09%)</title><rect x="917.7" y="517" width="1.0" height="15.0" fill="rgb(212,217,13)" rx="2" ry="2" />
<text  x="920.70" y="527.5" ></text>
</g>
<g >
<title>sync.(*Pool).getSlow (108 samples, 0.03%)</title><rect x="70.9" y="693" width="0.4" height="15.0" fill="rgb(244,106,0)" rx="2" ry="2" />
<text  x="73.91" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).WritePacket (278 samples, 0.09%)</title><rect x="181.0" y="517" width="1.0" height="15.0" fill="rgb(231,115,21)" rx="2" ry="2" />
<text  x="184.01" y="527.5" ></text>
</g>
<g >
<title>runtime.futex (69 samples, 0.02%)</title><rect x="1150.8" y="693" width="0.2" height="15.0" fill="rgb(228,25,31)" rx="2" ry="2" />
<text  x="1153.78" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).Enabled (51 samples, 0.02%)</title><rect x="787.3" y="661" width="0.1" height="15.0" fill="rgb(254,129,46)" rx="2" ry="2" />
<text  x="790.26" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*protocol).Parse (140 samples, 0.04%)</title><rect x="793.5" y="741" width="0.5" height="15.0" fill="rgb(220,104,36)" rx="2" ry="2" />
<text  x="796.51" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (546 samples, 0.17%)</title><rect x="1124.8" y="709" width="2.0" height="15.0" fill="rgb(254,190,6)" rx="2" ry="2" />
<text  x="1127.82" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (70 samples, 0.02%)</title><rect x="84.7" y="677" width="0.2" height="15.0" fill="rgb(226,220,41)" rx="2" ry="2" />
<text  x="87.69" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (68 samples, 0.02%)</title><rect x="1085.9" y="629" width="0.2" height="15.0" fill="rgb(242,104,13)" rx="2" ry="2" />
<text  x="1088.87" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (162 samples, 0.05%)</title><rect x="1187.0" y="629" width="0.6" height="15.0" fill="rgb(216,32,30)" rx="2" ry="2" />
<text  x="1190.03" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.newDirent (115 samples, 0.04%)</title><rect x="49.8" y="677" width="0.4" height="15.0" fill="rgb(236,224,47)" rx="2" ry="2" />
<text  x="52.82" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,582 samples, 1.40%)</title><rect x="805.6" y="757" width="16.6" height="15.0" fill="rgb(247,78,45)" rx="2" ry="2" />
<text  x="808.60" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (596 samples, 0.18%)</title><rect x="954.4" y="437" width="2.1" height="15.0" fill="rgb(209,26,1)" rx="2" ry="2" />
<text  x="957.39" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/buffer.(*VectorisedView).PullUp (29 samples, 0.01%)</title><rect x="845.3" y="805" width="0.1" height="15.0" fill="rgb(211,141,52)" rx="2" ry="2" />
<text  x="848.33" y="815.5" ></text>
</g>
<g >
<title>runtime.systemstack (85 samples, 0.03%)</title><rect x="1051.1" y="725" width="0.3" height="15.0" fill="rgb(231,25,50)" rx="2" ry="2" />
<text  x="1054.07" y="735.5" ></text>
</g>
<g >
<title>runtime.runqempty (123 samples, 0.04%)</title><rect x="156.4" y="645" width="0.4" height="15.0" fill="rgb(243,113,52)" rx="2" ry="2" />
<text  x="159.37" y="655.5" ></text>
</g>
<g >
<title>runtime.readvarint (53 samples, 0.02%)</title><rect x="1137.1" y="709" width="0.2" height="15.0" fill="rgb(205,151,40)" rx="2" ry="2" />
<text  x="1140.14" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).Activate (313 samples, 0.10%)</title><rect x="80.2" y="693" width="1.2" height="15.0" fill="rgb(251,19,31)" rx="2" ry="2" />
<text  x="83.24" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.(*IPv4).DestinationAddress (41 samples, 0.01%)</title><rect x="758.8" y="629" width="0.2" height="15.0" fill="rgb(218,211,4)" rx="2" ry="2" />
<text  x="761.85" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClock).GetTime (36 samples, 0.01%)</title><rect x="52.0" y="613" width="0.2" height="15.0" fill="rgb(217,158,42)" rx="2" ry="2" />
<text  x="55.02" y="623.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (89 samples, 0.03%)</title><rect x="50.7" y="613" width="0.3" height="15.0" fill="rgb(225,212,4)" rx="2" ry="2" />
<text  x="53.66" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.(*IPv4).SourceAddress (111 samples, 0.03%)</title><rect x="762.5" y="613" width="0.4" height="15.0" fill="rgb(208,213,21)" rx="2" ry="2" />
<text  x="765.49" y="623.5" ></text>
</g>
<g >
<title>runtime.startm (124 samples, 0.04%)</title><rect x="1028.3" y="485" width="0.5" height="15.0" fill="rgb(217,65,33)" rx="2" ry="2" />
<text  x="1031.33" y="495.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="956.4" y="373" width="0.1" height="15.0" fill="rgb(226,223,21)" rx="2" ry="2" />
<text  x="959.39" y="383.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyInTo (47 samples, 0.01%)</title><rect x="21.6" y="741" width="0.1" height="15.0" fill="rgb(219,3,51)" rx="2" ry="2" />
<text  x="24.57" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).LockUser (120 samples, 0.04%)</title><rect x="53.8" y="709" width="0.5" height="15.0" fill="rgb(223,173,41)" rx="2" ry="2" />
<text  x="56.82" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (402 samples, 0.12%)</title><rect x="196.4" y="277" width="1.4" height="15.0" fill="rgb(236,203,51)" rx="2" ry="2" />
<text  x="199.36" y="287.5" ></text>
</g>
<g >
<title>runtime.newobject (338 samples, 0.10%)</title><rect x="55.2" y="725" width="1.2" height="15.0" fill="rgb(229,152,41)" rx="2" ry="2" />
<text  x="58.20" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Get (31 samples, 0.01%)</title><rect x="203.6" y="757" width="0.1" height="15.0" fill="rgb(242,27,31)" rx="2" ry="2" />
<text  x="206.60" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="1062.4" y="549" width="0.1" height="15.0" fill="rgb(236,193,12)" rx="2" ry="2" />
<text  x="1065.37" y="559.5" ></text>
</g>
<g >
<title>runtime.runqgrab (50 samples, 0.02%)</title><rect x="29.2" y="853" width="0.2" height="15.0" fill="rgb(251,102,14)" rx="2" ry="2" />
<text  x="32.18" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).DeliverOutboundPacket (28 samples, 0.01%)</title><rect x="1073.9" y="517" width="0.1" height="15.0" fill="rgb(246,223,41)" rx="2" ry="2" />
<text  x="1076.86" y="527.5" ></text>
</g>
<g >
<title>runtime.mallocgc (31 samples, 0.01%)</title><rect x="202.6" y="661" width="0.1" height="15.0" fill="rgb(217,89,18)" rx="2" ry="2" />
<text  x="205.63" y="671.5" ></text>
</g>
<g >
<title>runtime.newobject (44 samples, 0.01%)</title><rect x="191.6" y="709" width="0.2" height="15.0" fill="rgb(247,59,46)" rx="2" ry="2" />
<text  x="194.61" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (350 samples, 0.11%)</title><rect x="1063.5" y="629" width="1.3" height="15.0" fill="rgb(208,120,12)" rx="2" ry="2" />
<text  x="1066.53" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (80 samples, 0.02%)</title><rect x="163.2" y="741" width="0.3" height="15.0" fill="rgb(215,128,9)" rx="2" ry="2" />
<text  x="166.21" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*protocol).Option (46 samples, 0.01%)</title><rect x="1009.7" y="741" width="0.2" height="15.0" fill="rgb(244,88,10)" rx="2" ry="2" />
<text  x="1012.69" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*IPTables).Check (58 samples, 0.02%)</title><rect x="739.3" y="709" width="0.2" height="15.0" fill="rgb(232,66,23)" rx="2" ry="2" />
<text  x="742.29" y="719.5" ></text>
</g>
<g >
<title>runtime.runtimer (30 samples, 0.01%)</title><rect x="97.3" y="629" width="0.1" height="15.0" fill="rgb(238,32,43)" rx="2" ry="2" />
<text  x="100.31" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).accountTaskGoroutineEnter (42 samples, 0.01%)</title><rect x="34.0" y="853" width="0.2" height="15.0" fill="rgb(219,3,8)" rx="2" ry="2" />
<text  x="37.02" y="863.5" ></text>
</g>
<g >
<title>runtime.adjusttimers (234 samples, 0.07%)</title><rect x="902.8" y="725" width="0.8" height="15.0" fill="rgb(253,143,39)" rx="2" ry="2" />
<text  x="905.76" y="735.5" ></text>
</g>
<g >
<title>runtime.futex (360 samples, 0.11%)</title><rect x="1065.2" y="693" width="1.3" height="15.0" fill="rgb(247,183,23)" rx="2" ry="2" />
<text  x="1068.22" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,463 samples, 2.29%)</title><rect x="967.6" y="405" width="27.0" height="15.0" fill="rgb(241,189,42)" rx="2" ry="2" />
<text  x="970.59" y="415.5" >[..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (786 samples, 0.24%)</title><rect x="124.5" y="501" width="2.9" height="15.0" fill="rgb(219,56,21)" rx="2" ry="2" />
<text  x="127.53" y="511.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (63 samples, 0.02%)</title><rect x="726.6" y="677" width="0.2" height="15.0" fill="rgb(238,148,46)" rx="2" ry="2" />
<text  x="729.62" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (373 samples, 0.11%)</title><rect x="756.7" y="325" width="1.4" height="15.0" fill="rgb(248,88,45)" rx="2" ry="2" />
<text  x="759.72" y="335.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleSynSegment (6,124 samples, 1.88%)</title><rect x="1022.1" y="837" width="22.2" height="15.0" fill="rgb(224,136,35)" rx="2" ry="2" />
<text  x="1025.11" y="847.5" >g..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (52,772 samples, 16.17%)</title><rect x="255.8" y="709" width="190.9" height="15.0" fill="rgb(210,139,16)" rx="2" ry="2" />
<text  x="258.81" y="719.5" >[[kernel.kallsyms]]</text>
</g>
<g >
<title>runtime.makeslice (66 samples, 0.02%)</title><rect x="206.8" y="661" width="0.2" height="15.0" fill="rgb(225,36,4)" rx="2" ry="2" />
<text  x="209.80" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,065 samples, 0.33%)</title><rect x="152.4" y="517" width="3.9" height="15.0" fill="rgb(234,114,46)" rx="2" ry="2" />
<text  x="155.44" y="527.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (5,361 samples, 1.64%)</title><rect x="802.8" y="789" width="19.4" height="15.0" fill="rgb(215,62,17)" rx="2" ry="2" />
<text  x="805.78" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).MainAddress.func1 (88 samples, 0.03%)</title><rect x="725.6" y="677" width="0.3" height="15.0" fill="rgb(206,54,46)" rx="2" ry="2" />
<text  x="728.56" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (74 samples, 0.02%)</title><rect x="1083.2" y="405" width="0.3" height="15.0" fill="rgb(251,61,21)" rx="2" ry="2" />
<text  x="1086.20" y="415.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (34 samples, 0.01%)</title><rect x="214.7" y="517" width="0.1" height="15.0" fill="rgb(242,107,54)" rx="2" ry="2" />
<text  x="217.72" y="527.5" ></text>
</g>
<g >
<title>runtime.newobject (65 samples, 0.02%)</title><rect x="1010.9" y="725" width="0.2" height="15.0" fill="rgb(222,36,10)" rx="2" ry="2" />
<text  x="1013.89" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.NewInode (286 samples, 0.09%)</title><rect x="50.3" y="693" width="1.0" height="15.0" fill="rgb(240,188,53)" rx="2" ry="2" />
<text  x="53.25" y="703.5" ></text>
</g>
<g >
<title>nft_do_chain_ipv4 (195 samples, 0.06%)</title><rect x="993.9" y="309" width="0.7" height="15.0" fill="rgb(253,80,15)" rx="2" ry="2" />
<text  x="996.87" y="319.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (111 samples, 0.03%)</title><rect x="1122.7" y="613" width="0.4" height="15.0" fill="rgb(245,11,26)" rx="2" ry="2" />
<text  x="1125.75" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*IPTables).Check (119 samples, 0.04%)</title><rect x="778.3" y="725" width="0.5" height="15.0" fill="rgb(226,133,36)" rx="2" ry="2" />
<text  x="781.35" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).setEndpointState (75 samples, 0.02%)</title><rect x="1008.5" y="773" width="0.2" height="15.0" fill="rgb(222,191,20)" rx="2" ry="2" />
<text  x="1011.45" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*globalDirentMap).add (58 samples, 0.02%)</title><rect x="49.6" y="677" width="0.2" height="15.0" fill="rgb(248,196,36)" rx="2" ry="2" />
<text  x="52.61" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (37 samples, 0.01%)</title><rect x="1150.9" y="581" width="0.1" height="15.0" fill="rgb(244,82,49)" rx="2" ry="2" />
<text  x="1153.89" y="591.5" ></text>
</g>
<g >
<title>runtime.newobject (61 samples, 0.02%)</title><rect x="52.7" y="693" width="0.3" height="15.0" fill="rgb(230,163,48)" rx="2" ry="2" />
<text  x="55.73" y="703.5" ></text>
</g>
<g >
<title>runtime.dodeltimer0 (69 samples, 0.02%)</title><rect x="113.8" y="581" width="0.3" height="15.0" fill="rgb(217,107,49)" rx="2" ry="2" />
<text  x="116.80" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*protocol).Option (121 samples, 0.04%)</title><rect x="1039.9" y="757" width="0.4" height="15.0" fill="rgb(254,24,1)" rx="2" ry="2" />
<text  x="1042.89" y="767.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (83 samples, 0.03%)</title><rect x="172.8" y="709" width="0.3" height="15.0" fill="rgb(249,147,14)" rx="2" ry="2" />
<text  x="175.77" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (60 samples, 0.02%)</title><rect x="989.5" y="85" width="0.2" height="15.0" fill="rgb(239,100,50)" rx="2" ry="2" />
<text  x="992.46" y="95.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (116 samples, 0.04%)</title><rect x="157.4" y="437" width="0.4" height="15.0" fill="rgb(231,198,16)" rx="2" ry="2" />
<text  x="160.40" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverOutboundPacket (42 samples, 0.01%)</title><rect x="1091.4" y="581" width="0.1" height="15.0" fill="rgb(236,14,35)" rx="2" ry="2" />
<text  x="1094.38" y="591.5" ></text>
</g>
<g >
<title>runtime.scanblock (1,132 samples, 0.35%)</title><rect x="1142.4" y="725" width="4.1" height="15.0" fill="rgb(253,196,4)" rx="2" ry="2" />
<text  x="1145.41" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.unrolledCalculateChecksum (37 samples, 0.01%)</title><rect x="1075.9" y="677" width="0.1" height="15.0" fill="rgb(246,46,8)" rx="2" ry="2" />
<text  x="1078.90" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).resolvedFields (153 samples, 0.05%)</title><rect x="181.5" y="485" width="0.5" height="15.0" fill="rgb(241,192,2)" rx="2" ry="2" />
<text  x="184.46" y="495.5" ></text>
</g>
<g >
<title>runtime.runtimer (133 samples, 0.04%)</title><rect x="1046.0" y="757" width="0.5" height="15.0" fill="rgb(239,84,21)" rx="2" ry="2" />
<text  x="1049.00" y="767.5" ></text>
</g>
<g >
<title>runtime.pcvalue (212 samples, 0.06%)</title><rect x="1096.2" y="677" width="0.8" height="15.0" fill="rgb(238,65,12)" rx="2" ry="2" />
<text  x="1099.19" y="687.5" ></text>
</g>
<g >
<title>runtime.findrunnable (42 samples, 0.01%)</title><rect x="696.8" y="789" width="0.2" height="15.0" fill="rgb(213,122,46)" rx="2" ry="2" />
<text  x="699.81" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="1062.3" y="565" width="0.2" height="15.0" fill="rgb(228,2,39)" rx="2" ry="2" />
<text  x="1065.35" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportEndpoints).unregisterEndpoint (1,120 samples, 0.34%)</title><rect x="1101.1" y="789" width="4.1" height="15.0" fill="rgb(235,80,34)" rx="2" ry="2" />
<text  x="1104.11" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (83 samples, 0.03%)</title><rect x="1062.2" y="645" width="0.3" height="15.0" fill="rgb(213,145,25)" rx="2" ry="2" />
<text  x="1065.20" y="655.5" ></text>
</g>
<g >
<title>runtime.casgstatus (29 samples, 0.01%)</title><rect x="446.7" y="741" width="0.1" height="15.0" fill="rgb(240,215,1)" rx="2" ry="2" />
<text  x="449.74" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*EventPoll).AddEntry (674 samples, 0.21%)</title><rect x="70.4" y="741" width="2.4" height="15.0" fill="rgb(253,3,10)" rx="2" ry="2" />
<text  x="73.40" y="751.5" ></text>
</g>
<g >
<title>runtime.makeslice (39 samples, 0.01%)</title><rect x="182.4" y="565" width="0.2" height="15.0" fill="rgb(253,38,19)" rx="2" ry="2" />
<text  x="185.44" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*EventPoll).initEntryReadiness (115 samples, 0.04%)</title><rect x="71.5" y="725" width="0.4" height="15.0" fill="rgb(238,121,44)" rx="2" ry="2" />
<text  x="74.47" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*sender).sendSegmentFromView (2,141 samples, 0.66%)</title><rect x="208.1" y="645" width="7.8" height="15.0" fill="rgb(246,211,0)" rx="2" ry="2" />
<text  x="211.15" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).writePacketBuffer (121 samples, 0.04%)</title><rect x="181.0" y="485" width="0.5" height="15.0" fill="rgb(208,164,31)" rx="2" ry="2" />
<text  x="184.02" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*protocol).QueuePacket (7,165 samples, 2.20%)</title><rect x="743.1" y="661" width="25.9" height="15.0" fill="rgb(217,213,3)" rx="2" ry="2" />
<text  x="746.08" y="671.5" >g..</text>
</g>
<g >
<title>runtime.pcdatavalue (178 samples, 0.05%)</title><rect x="1053.7" y="677" width="0.6" height="15.0" fill="rgb(242,187,37)" rx="2" ry="2" />
<text  x="1056.67" y="687.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (68 samples, 0.02%)</title><rect x="763.8" y="565" width="0.3" height="15.0" fill="rgb(219,100,47)" rx="2" ry="2" />
<text  x="766.84" y="575.5" ></text>
</g>
<g >
<title>runtime.mallocgc (68 samples, 0.02%)</title><rect x="182.6" y="549" width="0.3" height="15.0" fill="rgb(232,81,16)" rx="2" ry="2" />
<text  x="185.63" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (72 samples, 0.02%)</title><rect x="1091.6" y="597" width="0.3" height="15.0" fill="rgb(221,228,26)" rx="2" ry="2" />
<text  x="1094.60" y="607.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (60 samples, 0.02%)</title><rect x="51.0" y="645" width="0.3" height="15.0" fill="rgb(244,3,44)" rx="2" ry="2" />
<text  x="54.04" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (85 samples, 0.03%)</title><rect x="1049.4" y="517" width="0.3" height="15.0" fill="rgb(231,226,11)" rx="2" ry="2" />
<text  x="1052.43" y="527.5" ></text>
</g>
<g >
<title>runtime.read (76 samples, 0.02%)</title><rect x="1061.2" y="741" width="0.3" height="15.0" fill="rgb(212,136,30)" rx="2" ry="2" />
<text  x="1064.22" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportDemuxer).deliverPacket (92 samples, 0.03%)</title><rect x="19.7" y="789" width="0.3" height="15.0" fill="rgb(230,1,52)" rx="2" ry="2" />
<text  x="22.66" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).UninterruptibleSleepFinish (32 samples, 0.01%)</title><rect x="201.9" y="565" width="0.1" height="15.0" fill="rgb(252,104,37)" rx="2" ry="2" />
<text  x="204.85" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (197 samples, 0.06%)</title><rect x="921.2" y="549" width="0.7" height="15.0" fill="rgb(227,113,1)" rx="2" ry="2" />
<text  x="924.16" y="559.5" ></text>
</g>
<g >
<title>runtime.getStackMap (871 samples, 0.27%)</title><rect x="1139.2" y="725" width="3.1" height="15.0" fill="rgb(243,5,1)" rx="2" ry="2" />
<text  x="1142.16" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (58 samples, 0.02%)</title><rect x="28.9" y="789" width="0.3" height="15.0" fill="rgb(232,161,47)" rx="2" ry="2" />
<text  x="31.94" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (215 samples, 0.07%)</title><rect x="1065.7" y="597" width="0.8" height="15.0" fill="rgb(240,59,1)" rx="2" ry="2" />
<text  x="1068.75" y="607.5" ></text>
</g>
<g >
<title>runtime.newobject (49 samples, 0.02%)</title><rect x="1023.2" y="757" width="0.2" height="15.0" fill="rgb(233,212,52)" rx="2" ry="2" />
<text  x="1026.19" y="767.5" ></text>
</g>
<g >
<title>runtime.futex (64 samples, 0.02%)</title><rect x="15.9" y="821" width="0.2" height="15.0" fill="rgb(227,206,40)" rx="2" ry="2" />
<text  x="18.89" y="831.5" ></text>
</g>
<g >
<title>runtime.futex (408 samples, 0.13%)</title><rect x="20.0" y="869" width="1.5" height="15.0" fill="rgb(241,198,54)" rx="2" ry="2" />
<text  x="23.04" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (70 samples, 0.02%)</title><rect x="1062.2" y="597" width="0.3" height="15.0" fill="rgb(234,84,36)" rx="2" ry="2" />
<text  x="1065.25" y="607.5" ></text>
</g>
<g >
<title>syscall.Syscall6 (2,279 samples, 0.70%)</title><rect x="193.6" y="501" width="8.2" height="15.0" fill="rgb(242,70,0)" rx="2" ry="2" />
<text  x="196.58" y="511.5" ></text>
</g>
<g >
<title>runtime.mallocgc (36 samples, 0.01%)</title><rect x="214.1" y="469" width="0.1" height="15.0" fill="rgb(227,138,36)" rx="2" ry="2" />
<text  x="217.06" y="479.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (245 samples, 0.08%)</title><rect x="115.0" y="549" width="0.9" height="15.0" fill="rgb(230,83,39)" rx="2" ry="2" />
<text  x="117.99" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).Capabilities (44 samples, 0.01%)</title><rect x="719.7" y="757" width="0.2" height="15.0" fill="rgb(218,111,40)" rx="2" ry="2" />
<text  x="722.70" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withVecInternalMappings (2,781 samples, 0.85%)</title><rect x="192.5" y="661" width="10.1" height="15.0" fill="rgb(221,123,4)" rx="2" ry="2" />
<text  x="195.50" y="671.5" ></text>
</g>
<g >
<title>time.(*Timer).Reset (48 samples, 0.01%)</title><rect x="1088.6" y="773" width="0.1" height="15.0" fill="rgb(232,155,9)" rx="2" ry="2" />
<text  x="1091.57" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (80 samples, 0.02%)</title><rect x="904.6" y="613" width="0.3" height="15.0" fill="rgb(206,71,29)" rx="2" ry="2" />
<text  x="907.61" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).DeliverNetworkPacket (21,414 samples, 6.56%)</title><rect x="719.0" y="789" width="77.4" height="15.0" fill="rgb(239,123,7)" rx="2" ry="2" />
<text  x="722.01" y="799.5" >gvisor.d..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.generateSecureISN (49 samples, 0.02%)</title><rect x="1043.1" y="805" width="0.2" height="15.0" fill="rgb(240,209,35)" rx="2" ry="2" />
<text  x="1046.09" y="815.5" ></text>
</g>
<g >
<title>runtime.pcvalue (50 samples, 0.02%)</title><rect x="1184.1" y="773" width="0.2" height="15.0" fill="rgb(210,100,17)" rx="2" ry="2" />
<text  x="1187.08" y="783.5" ></text>
</g>
<g >
<title>runtime.gcDrainN (56 samples, 0.02%)</title><rect x="1020.2" y="757" width="0.2" height="15.0" fill="rgb(211,10,2)" rx="2" ry="2" />
<text  x="1023.16" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*segment).parse (263 samples, 0.08%)</title><rect x="760.4" y="629" width="1.0" height="15.0" fill="rgb(245,15,34)" rx="2" ry="2" />
<text  x="763.43" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (398 samples, 0.12%)</title><rect x="705.3" y="613" width="1.4" height="15.0" fill="rgb(209,81,3)" rx="2" ry="2" />
<text  x="708.26" y="623.5" ></text>
</g>
<g >
<title>runtime.procyield (208 samples, 0.06%)</title><rect x="112.8" y="613" width="0.7" height="15.0" fill="rgb(207,123,41)" rx="2" ry="2" />
<text  x="115.78" y="623.5" ></text>
</g>
<g >
<title>runtime.getitab (76 samples, 0.02%)</title><rect x="792.2" y="693" width="0.3" height="15.0" fill="rgb(234,179,50)" rx="2" ry="2" />
<text  x="795.20" y="703.5" ></text>
</g>
<g >
<title>runtime.stackcacherelease (118 samples, 0.04%)</title><rect x="1153.6" y="725" width="0.4" height="15.0" fill="rgb(242,104,12)" rx="2" ry="2" />
<text  x="1156.56" y="735.5" ></text>
</g>
<g >
<title>runtime.copystack (921 samples, 0.28%)</title><rect x="1052.1" y="741" width="3.3" height="15.0" fill="rgb(234,225,53)" rx="2" ry="2" />
<text  x="1055.06" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyOutBytes (69 samples, 0.02%)</title><rect x="57.2" y="693" width="0.2" height="15.0" fill="rgb(231,9,22)" rx="2" ry="2" />
<text  x="60.16" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (329 samples, 0.10%)</title><rect x="1122.0" y="709" width="1.1" height="15.0" fill="rgb(233,229,30)" rx="2" ry="2" />
<text  x="1124.96" y="719.5" ></text>
</g>
<g >
<title>runtime.futex (68 samples, 0.02%)</title><rect x="1189.7" y="773" width="0.3" height="15.0" fill="rgb(245,133,29)" rx="2" ry="2" />
<text  x="1192.75" y="783.5" ></text>
</g>
<g >
<title>runtime.interhash (37 samples, 0.01%)</title><rect x="217.9" y="757" width="0.1" height="15.0" fill="rgb(245,199,29)" rx="2" ry="2" />
<text  x="220.87" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (41 samples, 0.01%)</title><rect x="1150.9" y="597" width="0.1" height="15.0" fill="rgb(207,85,14)" rx="2" ry="2" />
<text  x="1153.88" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (55 samples, 0.02%)</title><rect x="766.3" y="453" width="0.2" height="15.0" fill="rgb(240,110,1)" rx="2" ry="2" />
<text  x="769.29" y="463.5" ></text>
</g>
<g >
<title>runtime.funcspdelta (115 samples, 0.04%)</title><rect x="1104.1" y="709" width="0.4" height="15.0" fill="rgb(224,146,42)" rx="2" ry="2" />
<text  x="1107.07" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.FromIOWriter.writeFromBlock (47 samples, 0.01%)</title><rect x="21.6" y="613" width="0.1" height="15.0" fill="rgb(238,64,53)" rx="2" ry="2" />
<text  x="24.57" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).enqueuePacketBuffer (1,206 samples, 0.37%)</title><rect x="209.9" y="517" width="4.3" height="15.0" fill="rgb(205,211,44)" rx="2" ry="2" />
<text  x="212.88" y="527.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,570 samples, 0.48%)</title><rect x="142.7" y="421" width="5.7" height="15.0" fill="rgb(211,103,47)" rx="2" ry="2" />
<text  x="145.74" y="431.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (135 samples, 0.04%)</title><rect x="1187.1" y="597" width="0.5" height="15.0" fill="rgb(212,53,2)" rx="2" ry="2" />
<text  x="1190.13" y="607.5" ></text>
</g>
<g >
<title>runtime.growslice (1,216 samples, 0.37%)</title><rect x="712.9" y="805" width="4.4" height="15.0" fill="rgb(215,88,2)" rx="2" ry="2" />
<text  x="715.88" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.AddressWithPrefix.Subnet (55 samples, 0.02%)</title><rect x="1033.5" y="725" width="0.2" height="15.0" fill="rgb(248,22,3)" rx="2" ry="2" />
<text  x="1036.45" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,301 samples, 0.71%)</title><rect x="832.9" y="629" width="8.3" height="15.0" fill="rgb(248,193,3)" rx="2" ry="2" />
<text  x="835.93" y="639.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (37 samples, 0.01%)</title><rect x="1074.1" y="501" width="0.1" height="15.0" fill="rgb(235,88,0)" rx="2" ry="2" />
<text  x="1077.08" y="511.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (149 samples, 0.05%)</title><rect x="730.5" y="693" width="0.6" height="15.0" fill="rgb(225,102,35)" rx="2" ry="2" />
<text  x="733.55" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).GetKind (36 samples, 0.01%)</title><rect x="1036.9" y="661" width="0.1" height="15.0" fill="rgb(222,52,33)" rx="2" ry="2" />
<text  x="1039.91" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (75 samples, 0.02%)</title><rect x="1074.0" y="549" width="0.3" height="15.0" fill="rgb(236,177,51)" rx="2" ry="2" />
<text  x="1077.02" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.ContextCanAccessFile (175 samples, 0.05%)</title><rect x="188.2" y="629" width="0.6" height="15.0" fill="rgb(223,140,40)" rx="2" ry="2" />
<text  x="191.16" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.sendTCP (1,255 samples, 0.38%)</title><rect x="1089.9" y="773" width="4.6" height="15.0" fill="rgb(215,125,14)" rx="2" ry="2" />
<text  x="1092.94" y="783.5" ></text>
</g>
<g >
<title>runtime.stopm (87 samples, 0.03%)</title><rect x="17.9" y="789" width="0.3" height="15.0" fill="rgb(226,98,41)" rx="2" ry="2" />
<text  x="20.87" y="799.5" ></text>
</g>
<g >
<title>runtime.park_m (44 samples, 0.01%)</title><rect x="696.8" y="821" width="0.2" height="15.0" fill="rgb(236,221,12)" rx="2" ry="2" />
<text  x="699.81" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,700 samples, 0.52%)</title><rect x="751.9" y="453" width="6.2" height="15.0" fill="rgb(220,203,47)" rx="2" ry="2" />
<text  x="754.92" y="463.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (43 samples, 0.01%)</title><rect x="207.4" y="613" width="0.2" height="15.0" fill="rgb(209,171,17)" rx="2" ry="2" />
<text  x="210.41" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (14,656 samples, 4.49%)</title><rect x="943.7" y="645" width="53.0" height="15.0" fill="rgb(221,55,16)" rx="2" ry="2" />
<text  x="946.71" y="655.5" >[[ker..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (47 samples, 0.01%)</title><rect x="1079.0" y="661" width="0.2" height="15.0" fill="rgb(250,224,44)" rx="2" ry="2" />
<text  x="1082.05" y="671.5" ></text>
</g>
<g >
<title>__nf_ct_l4proto_find (91 samples, 0.03%)</title><rect x="961.1" y="453" width="0.3" height="15.0" fill="rgb(247,132,41)" rx="2" ry="2" />
<text  x="964.12" y="463.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*segmentQueue).dequeue (28 samples, 0.01%)</title><rect x="1099.5" y="837" width="0.1" height="15.0" fill="rgb(253,226,51)" rx="2" ry="2" />
<text  x="1102.47" y="847.5" ></text>
</g>
<g >
<title>runtime.acquireSudog (188 samples, 0.06%)</title><rect x="90.8" y="693" width="0.7" height="15.0" fill="rgb(229,159,5)" rx="2" ry="2" />
<text  x="93.80" y="703.5" ></text>
</g>
<g >
<title>fmt.(*pp).doPrintf (168 samples, 0.05%)</title><rect x="48.5" y="677" width="0.6" height="15.0" fill="rgb(208,174,10)" rx="2" ry="2" />
<text  x="51.50" y="687.5" ></text>
</g>
<g >
<title>reflect.Value.NumField (41 samples, 0.01%)</title><rect x="925.8" y="789" width="0.1" height="15.0" fill="rgb(228,96,6)" rx="2" ry="2" />
<text  x="928.76" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (90 samples, 0.03%)</title><rect x="212.8" y="117" width="0.3" height="15.0" fill="rgb(213,120,9)" rx="2" ry="2" />
<text  x="215.79" y="127.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (83 samples, 0.03%)</title><rect x="1007.4" y="677" width="0.3" height="15.0" fill="rgb(227,1,14)" rx="2" ry="2" />
<text  x="1010.36" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (287 samples, 0.09%)</title><rect x="199.8" y="277" width="1.1" height="15.0" fill="rgb(227,212,13)" rx="2" ry="2" />
<text  x="202.85" y="287.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (576 samples, 0.18%)</title><rect x="756.0" y="341" width="2.1" height="15.0" fill="rgb(239,46,23)" rx="2" ry="2" />
<text  x="758.99" y="351.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (447 samples, 0.14%)</title><rect x="59.3" y="741" width="1.6" height="15.0" fill="rgb(226,176,5)" rx="2" ry="2" />
<text  x="62.29" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (141 samples, 0.04%)</title><rect x="1128.1" y="629" width="0.5" height="15.0" fill="rgb(227,63,32)" rx="2" ry="2" />
<text  x="1131.05" y="639.5" ></text>
</g>
<g >
<title>runtime.systemstack (44 samples, 0.01%)</title><rect x="1069.5" y="805" width="0.1" height="15.0" fill="rgb(239,3,3)" rx="2" ry="2" />
<text  x="1072.47" y="815.5" ></text>
</g>
<g >
<title>runtime.getStackMap (344 samples, 0.11%)</title><rect x="1095.8" y="709" width="1.2" height="15.0" fill="rgb(248,94,31)" rx="2" ry="2" />
<text  x="1098.76" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).writePacket (523 samples, 0.16%)</title><rect x="1073.4" y="677" width="1.9" height="15.0" fill="rgb(220,157,28)" rx="2" ry="2" />
<text  x="1076.44" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindNICNameFromID (106 samples, 0.03%)</title><rect x="775.8" y="709" width="0.4" height="15.0" fill="rgb(247,164,2)" rx="2" ry="2" />
<text  x="778.79" y="719.5" ></text>
</g>
<g >
<title>runtime.addtimer (57 samples, 0.02%)</title><rect x="1040.9" y="725" width="0.2" height="15.0" fill="rgb(244,42,46)" rx="2" ry="2" />
<text  x="1043.87" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (71 samples, 0.02%)</title><rect x="198.9" y="261" width="0.3" height="15.0" fill="rgb(246,83,47)" rx="2" ry="2" />
<text  x="201.92" y="271.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (3,234 samples, 0.99%)</title><rect x="980.9" y="229" width="11.7" height="15.0" fill="rgb(214,95,10)" rx="2" ry="2" />
<text  x="983.90" y="239.5" ></text>
</g>
<g >
<title>runtime.acquirep (42 samples, 0.01%)</title><rect x="911.0" y="725" width="0.2" height="15.0" fill="rgb(252,123,39)" rx="2" ry="2" />
<text  x="914.05" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (152 samples, 0.05%)</title><rect x="451.7" y="533" width="0.5" height="15.0" fill="rgb(233,222,36)" rx="2" ry="2" />
<text  x="454.69" y="543.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="1109.2" y="661" width="0.1" height="15.0" fill="rgb(227,151,39)" rx="2" ry="2" />
<text  x="1112.18" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (50 samples, 0.02%)</title><rect x="22.4" y="741" width="0.2" height="15.0" fill="rgb(244,86,15)" rx="2" ry="2" />
<text  x="25.43" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (218 samples, 0.07%)</title><rect x="1188.9" y="613" width="0.8" height="15.0" fill="rgb(246,204,2)" rx="2" ry="2" />
<text  x="1191.92" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).AcquireAssignedAddressOrMatching (53 samples, 0.02%)</title><rect x="791.2" y="709" width="0.2" height="15.0" fill="rgb(239,42,16)" rx="2" ry="2" />
<text  x="794.21" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindNICNameFromID (33 samples, 0.01%)</title><rect x="1092.9" y="725" width="0.2" height="15.0" fill="rgb(242,178,8)" rx="2" ry="2" />
<text  x="1095.93" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyOutBytes (34 samples, 0.01%)</title><rect x="57.5" y="725" width="0.1" height="15.0" fill="rgb(227,36,36)" rx="2" ry="2" />
<text  x="60.45" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (31 samples, 0.01%)</title><rect x="1150.9" y="549" width="0.1" height="15.0" fill="rgb(225,48,43)" rx="2" ry="2" />
<text  x="1153.91" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).getOrCreateEntry (52 samples, 0.02%)</title><rect x="1074.7" y="581" width="0.2" height="15.0" fill="rgb(207,117,1)" rx="2" ry="2" />
<text  x="1077.75" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborEntry).handleUpperLevelConfirmationLocked (2,499 samples, 0.77%)</title><rect x="1047.0" y="821" width="9.0" height="15.0" fill="rgb(221,11,39)" rx="2" ry="2" />
<text  x="1049.96" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (541 samples, 0.17%)</title><rect x="196.0" y="341" width="1.9" height="15.0" fill="rgb(218,60,39)" rx="2" ry="2" />
<text  x="198.97" y="351.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).Notify (763 samples, 0.23%)</title><rect x="1001.3" y="805" width="2.8" height="15.0" fill="rgb(209,81,51)" rx="2" ry="2" />
<text  x="1004.29" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).InotifyEvent (37 samples, 0.01%)</title><rect x="204.7" y="741" width="0.1" height="15.0" fill="rgb(221,168,28)" rx="2" ry="2" />
<text  x="207.67" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (54 samples, 0.02%)</title><rect x="452.3" y="597" width="0.2" height="15.0" fill="rgb(224,174,38)" rx="2" ry="2" />
<text  x="455.31" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).Broadcast (152 samples, 0.05%)</title><rect x="728.8" y="693" width="0.5" height="15.0" fill="rgb(241,213,9)" rx="2" ry="2" />
<text  x="731.79" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).Notify (44 samples, 0.01%)</title><rect x="1105.5" y="853" width="0.1" height="15.0" fill="rgb(225,200,1)" rx="2" ry="2" />
<text  x="1108.46" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).destroy (199 samples, 0.06%)</title><rect x="63.9" y="661" width="0.8" height="15.0" fill="rgb(210,198,23)" rx="2" ry="2" />
<text  x="66.93" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="199.5" y="277" width="0.1" height="15.0" fill="rgb(213,124,15)" rx="2" ry="2" />
<text  x="202.49" y="287.5" ></text>
</g>
<g >
<title>runtime.isSystemGoroutine (59 samples, 0.02%)</title><rect x="1044.0" y="773" width="0.2" height="15.0" fill="rgb(208,86,53)" rx="2" ry="2" />
<text  x="1046.97" y="783.5" ></text>
</g>
<g >
<title>runtime.newobject (113 samples, 0.03%)</title><rect x="711.8" y="805" width="0.4" height="15.0" fill="rgb(232,197,19)" rx="2" ry="2" />
<text  x="714.82" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (401 samples, 0.12%)</title><rect x="705.2" y="645" width="1.5" height="15.0" fill="rgb(208,52,52)" rx="2" ry="2" />
<text  x="708.25" y="655.5" ></text>
</g>
<g >
<title>runtime.getStackMap (295 samples, 0.09%)</title><rect x="1053.3" y="693" width="1.0" height="15.0" fill="rgb(218,64,14)" rx="2" ry="2" />
<text  x="1056.26" y="703.5" ></text>
</g>
<g >
<title>runtime.scanobject (39 samples, 0.01%)</title><rect x="1183.4" y="837" width="0.1" height="15.0" fill="rgb(251,9,38)" rx="2" ry="2" />
<text  x="1186.35" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (118 samples, 0.04%)</title><rect x="1108.6" y="709" width="0.4" height="15.0" fill="rgb(218,167,5)" rx="2" ry="2" />
<text  x="1111.61" y="719.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (33 samples, 0.01%)</title><rect x="1091.7" y="549" width="0.1" height="15.0" fill="rgb(236,41,46)" rx="2" ry="2" />
<text  x="1094.70" y="559.5" ></text>
</g>
<g >
<title>runtime.goready (37 samples, 0.01%)</title><rect x="758.5" y="613" width="0.1" height="15.0" fill="rgb(213,21,36)" rx="2" ry="2" />
<text  x="761.49" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (41 samples, 0.01%)</title><rect x="699.9" y="741" width="0.2" height="15.0" fill="rgb(219,83,10)" rx="2" ry="2" />
<text  x="702.93" y="751.5" ></text>
</g>
<g >
<title>time.goFunc (64 samples, 0.02%)</title><rect x="904.0" y="693" width="0.2" height="15.0" fill="rgb(238,97,43)" rx="2" ry="2" />
<text  x="906.99" y="703.5" ></text>
</g>
<g >
<title>runtime.evacuate (81 samples, 0.02%)</title><rect x="1025.9" y="741" width="0.2" height="15.0" fill="rgb(247,60,6)" rx="2" ry="2" />
<text  x="1028.85" y="751.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (45 samples, 0.01%)</title><rect x="183.2" y="661" width="0.1" height="15.0" fill="rgb(221,23,21)" rx="2" ry="2" />
<text  x="186.18" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).findLocalRouteFromNICRLocked (924 samples, 0.28%)</title><rect x="1034.6" y="757" width="3.3" height="15.0" fill="rgb(215,56,47)" rx="2" ry="2" />
<text  x="1037.58" y="767.5" ></text>
</g>
<g >
<title>runtime.memmove (53 samples, 0.02%)</title><rect x="1099.0" y="757" width="0.2" height="15.0" fill="rgb(244,3,34)" rx="2" ry="2" />
<text  x="1102.05" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (181 samples, 0.06%)</title><rect x="1130.2" y="709" width="0.7" height="15.0" fill="rgb(218,102,2)" rx="2" ry="2" />
<text  x="1133.25" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (104 samples, 0.03%)</title><rect x="23.3" y="741" width="0.4" height="15.0" fill="rgb(225,162,5)" rx="2" ry="2" />
<text  x="26.33" y="751.5" ></text>
</g>
<g >
<title>runtime.park_m (41 samples, 0.01%)</title><rect x="158.7" y="693" width="0.1" height="15.0" fill="rgb(223,79,34)" rx="2" ry="2" />
<text  x="161.67" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Write (3,065 samples, 0.94%)</title><rect x="205.3" y="709" width="11.1" height="15.0" fill="rgb(206,2,10)" rx="2" ry="2" />
<text  x="208.31" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).Notify (1,068 samples, 0.33%)</title><rect x="1000.4" y="837" width="3.8" height="15.0" fill="rgb(235,199,29)" rx="2" ry="2" />
<text  x="1003.35" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (123 samples, 0.04%)</title><rect x="1187.2" y="581" width="0.4" height="15.0" fill="rgb(218,42,23)" rx="2" ry="2" />
<text  x="1190.17" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).maybeExtendReference (44 samples, 0.01%)</title><rect x="187.7" y="709" width="0.2" height="15.0" fill="rgb(252,176,49)" rx="2" ry="2" />
<text  x="190.74" y="719.5" ></text>
</g>
<g >
<title>runtime.adjusttimers (35 samples, 0.01%)</title><rect x="1113.8" y="805" width="0.1" height="15.0" fill="rgb(253,60,22)" rx="2" ry="2" />
<text  x="1116.79" y="815.5" ></text>
</g>
<g >
<title>runtime.step (160 samples, 0.05%)</title><rect x="1017.2" y="693" width="0.6" height="15.0" fill="rgb(233,204,36)" rx="2" ry="2" />
<text  x="1020.23" y="703.5" ></text>
</g>
<g >
<title>runtime.park_m (32 samples, 0.01%)</title><rect x="52.6" y="677" width="0.1" height="15.0" fill="rgb(216,110,48)" rx="2" ry="2" />
<text  x="55.61" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (295 samples, 0.09%)</title><rect x="14.8" y="677" width="1.1" height="15.0" fill="rgb(230,228,51)" rx="2" ry="2" />
<text  x="17.80" y="687.5" ></text>
</g>
<g >
<title>runtime.adjustframe (511 samples, 0.16%)</title><rect x="1052.5" y="709" width="1.8" height="15.0" fill="rgb(217,134,12)" rx="2" ry="2" />
<text  x="1055.48" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (101 samples, 0.03%)</title><rect x="198.8" y="293" width="0.4" height="15.0" fill="rgb(213,34,34)" rx="2" ry="2" />
<text  x="201.81" y="303.5" ></text>
</g>
<g >
<title>runtime.funcspdelta (476 samples, 0.15%)</title><rect x="1135.7" y="757" width="1.7" height="15.0" fill="rgb(220,152,49)" rx="2" ry="2" />
<text  x="1138.72" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (123 samples, 0.04%)</title><rect x="1007.2" y="693" width="0.5" height="15.0" fill="rgb(251,112,0)" rx="2" ry="2" />
<text  x="1010.21" y="703.5" ></text>
</g>
<g >
<title>runtime.memclrNoHeapPointers (64 samples, 0.02%)</title><rect x="1042.2" y="677" width="0.2" height="15.0" fill="rgb(251,90,29)" rx="2" ry="2" />
<text  x="1045.20" y="687.5" ></text>
</g>
<g >
<title>runtime.futex (707 samples, 0.22%)</title><rect x="919.3" y="693" width="2.6" height="15.0" fill="rgb(251,37,46)" rx="2" ry="2" />
<text  x="922.32" y="703.5" ></text>
</g>
<g >
<title>runtime.newobject (49 samples, 0.02%)</title><rect x="214.0" y="485" width="0.2" height="15.0" fill="rgb(228,24,50)" rx="2" ry="2" />
<text  x="217.03" y="495.5" ></text>
</g>
<g >
<title>runtime.runqget (50 samples, 0.02%)</title><rect x="118.2" y="629" width="0.2" height="15.0" fill="rgb(220,127,1)" rx="2" ry="2" />
<text  x="121.18" y="639.5" ></text>
</g>
<g >
<title>__br_forward (264 samples, 0.08%)</title><rect x="989.9" y="149" width="0.9" height="15.0" fill="rgb(247,166,17)" rx="2" ry="2" />
<text  x="992.87" y="159.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.BlockSeq.Head (34 samples, 0.01%)</title><rect x="74.5" y="677" width="0.2" height="15.0" fill="rgb(252,190,35)" rx="2" ry="2" />
<text  x="77.54" y="687.5" ></text>
</g>
<g >
<title>runtime.(*pageAlloc).scavengeOne (233 samples, 0.07%)</title><rect x="1130.1" y="821" width="0.8" height="15.0" fill="rgb(241,216,11)" rx="2" ry="2" />
<text  x="1133.09" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (602 samples, 0.18%)</title><rect x="908.6" y="629" width="2.2" height="15.0" fill="rgb(243,8,48)" rx="2" ry="2" />
<text  x="911.60" y="639.5" ></text>
</g>
<g >
<title>runtime.duffcopy (46 samples, 0.01%)</title><rect x="780.0" y="725" width="0.1" height="15.0" fill="rgb(219,180,42)" rx="2" ry="2" />
<text  x="782.96" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (145 samples, 0.04%)</title><rect x="1049.2" y="597" width="0.5" height="15.0" fill="rgb(246,160,11)" rx="2" ry="2" />
<text  x="1052.22" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="990.0" y="117" width="0.1" height="15.0" fill="rgb(253,192,8)" rx="2" ry="2" />
<text  x="992.96" y="127.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).wait (61,314 samples, 18.79%)</title><rect x="229.5" y="805" width="221.7" height="15.0" fill="rgb(214,168,48)" rx="2" ry="2" />
<text  x="232.50" y="815.5" >gvisor.dev/gvisor/pkg/sentry/..</text>
</g>
<g >
<title>runtime.readvarint (37 samples, 0.01%)</title><rect x="1017.7" y="677" width="0.1" height="15.0" fill="rgb(234,187,29)" rx="2" ry="2" />
<text  x="1020.68" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (48 samples, 0.01%)</title><rect x="724.8" y="709" width="0.2" height="15.0" fill="rgb(211,224,9)" rx="2" ry="2" />
<text  x="727.82" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (229 samples, 0.07%)</title><rect x="997.3" y="629" width="0.8" height="15.0" fill="rgb(252,154,52)" rx="2" ry="2" />
<text  x="1000.30" y="639.5" ></text>
</g>
<g >
<title>runtime.mcall (429 samples, 0.13%)</title><rect x="705.2" y="821" width="1.5" height="15.0" fill="rgb(233,152,37)" rx="2" ry="2" />
<text  x="708.15" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (72 samples, 0.02%)</title><rect x="1128.3" y="597" width="0.3" height="15.0" fill="rgb(252,157,49)" rx="2" ry="2" />
<text  x="1131.30" y="607.5" ></text>
</g>
<g >
<title>runtime.copystack (317 samples, 0.10%)</title><rect x="1183.6" y="853" width="1.2" height="15.0" fill="rgb(220,165,8)" rx="2" ry="2" />
<text  x="1186.64" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (55 samples, 0.02%)</title><rect x="1049.5" y="469" width="0.2" height="15.0" fill="rgb(232,101,3)" rx="2" ry="2" />
<text  x="1052.54" y="479.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn.func1 (55 samples, 0.02%)</title><rect x="69.8" y="693" width="0.2" height="15.0" fill="rgb(239,229,36)" rx="2" ry="2" />
<text  x="72.76" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (120 samples, 0.04%)</title><rect x="904.5" y="677" width="0.4" height="15.0" fill="rgb(205,62,20)" rx="2" ry="2" />
<text  x="907.47" y="687.5" ></text>
</g>
<g >
<title>syscall.wait4 (78 samples, 0.02%)</title><rect x="450.8" y="773" width="0.3" height="15.0" fill="rgb(226,222,13)" rx="2" ry="2" />
<text  x="453.79" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (90 samples, 0.03%)</title><rect x="19.7" y="725" width="0.3" height="15.0" fill="rgb(233,84,1)" rx="2" ry="2" />
<text  x="22.66" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (74 samples, 0.02%)</title><rect x="1007.4" y="613" width="0.3" height="15.0" fill="rgb(228,74,47)" rx="2" ry="2" />
<text  x="1010.39" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="699.9" y="709" width="0.2" height="15.0" fill="rgb(224,15,22)" rx="2" ry="2" />
<text  x="702.94" y="719.5" ></text>
</g>
<g >
<title>runtime.morestack (1,574 samples, 0.48%)</title><rect x="1013.5" y="789" width="5.7" height="15.0" fill="rgb(248,184,20)" rx="2" ry="2" />
<text  x="1016.47" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).MaxHeaderLength (69 samples, 0.02%)</title><rect x="1072.8" y="693" width="0.3" height="15.0" fill="rgb(232,46,41)" rx="2" ry="2" />
<text  x="1075.84" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).MTU (67 samples, 0.02%)</title><rect x="1009.3" y="741" width="0.3" height="15.0" fill="rgb(212,193,10)" rx="2" ry="2" />
<text  x="1012.34" y="751.5" ></text>
</g>
<g >
<title>runtime.findrunnable (3,581 samples, 1.10%)</title><rect x="1114.2" y="821" width="12.9" height="15.0" fill="rgb(225,175,26)" rx="2" ry="2" />
<text  x="1117.17" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/usermem.AddrRangeSeq.DropFirst (49 samples, 0.02%)</title><rect x="206.5" y="613" width="0.2" height="15.0" fill="rgb(208,198,51)" rx="2" ry="2" />
<text  x="209.48" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (470 samples, 0.14%)</title><rect x="211.4" y="261" width="1.7" height="15.0" fill="rgb(241,168,32)" rx="2" ry="2" />
<text  x="214.41" y="271.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (68 samples, 0.02%)</title><rect x="1066.3" y="517" width="0.2" height="15.0" fill="rgb(248,203,2)" rx="2" ry="2" />
<text  x="1069.28" y="527.5" ></text>
</g>
<g >
<title>runtime.newobject (366 samples, 0.11%)</title><rect x="1012.0" y="757" width="1.3" height="15.0" fill="rgb(237,101,53)" rx="2" ry="2" />
<text  x="1015.01" y="767.5" ></text>
</g>
<g >
<title>runtime.gentraceback (984 samples, 0.30%)</title><rect x="1014.3" y="741" width="3.6" height="15.0" fill="rgb(211,42,50)" rx="2" ry="2" />
<text  x="1017.32" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).Activate (30 samples, 0.01%)</title><rect x="82.8" y="693" width="0.2" height="15.0" fill="rgb(206,99,41)" rx="2" ry="2" />
<text  x="85.84" y="703.5" ></text>
</g>
<g >
<title>runtime.write1 (242 samples, 0.07%)</title><rect x="1082.6" y="581" width="0.9" height="15.0" fill="rgb(216,189,25)" rx="2" ry="2" />
<text  x="1085.59" y="591.5" ></text>
</g>
<g >
<title>runtime.(*pageAlloc).allocRange (42 samples, 0.01%)</title><rect x="1098.4" y="661" width="0.1" height="15.0" fill="rgb(206,18,1)" rx="2" ry="2" />
<text  x="1101.39" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (425 samples, 0.13%)</title><rect x="786.8" y="677" width="1.6" height="15.0" fill="rgb(217,60,20)" rx="2" ry="2" />
<text  x="789.82" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (406 samples, 0.12%)</title><rect x="705.2" y="693" width="1.5" height="15.0" fill="rgb(228,149,37)" rx="2" ry="2" />
<text  x="708.23" y="703.5" ></text>
</g>
<g >
<title>runtime.stackcacherelease (45 samples, 0.01%)</title><rect x="1104.6" y="709" width="0.2" height="15.0" fill="rgb(223,189,34)" rx="2" ry="2" />
<text  x="1107.65" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,671 samples, 0.82%)</title><rect x="831.6" y="677" width="9.6" height="15.0" fill="rgb(244,132,15)" rx="2" ry="2" />
<text  x="834.59" y="687.5" ></text>
</g>
<g >
<title>runtime.profilealloc (52 samples, 0.02%)</title><rect x="767.9" y="581" width="0.2" height="15.0" fill="rgb(226,132,31)" rx="2" ry="2" />
<text  x="770.92" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (927 samples, 0.28%)</title><rect x="124.0" y="549" width="3.4" height="15.0" fill="rgb(231,110,52)" rx="2" ry="2" />
<text  x="127.02" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Unlock (46 samples, 0.01%)</title><rect x="736.2" y="645" width="0.2" height="15.0" fill="rgb(216,224,39)" rx="2" ry="2" />
<text  x="739.19" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).resetKickerLocked (32 samples, 0.01%)</title><rect x="1047.7" y="709" width="0.1" height="15.0" fill="rgb(247,68,23)" rx="2" ry="2" />
<text  x="1050.72" y="719.5" ></text>
</g>
<g >
<title>runtime.wakep (66 samples, 0.02%)</title><rect x="28.9" y="853" width="0.3" height="15.0" fill="rgb(234,225,38)" rx="2" ry="2" />
<text  x="31.94" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (16,992 samples, 5.21%)</title><rect x="936.7" y="725" width="61.4" height="15.0" fill="rgb(242,217,32)" rx="2" ry="2" />
<text  x="939.68" y="735.5" >[[kern..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).getRegs (28 samples, 0.01%)</title><rect x="229.2" y="805" width="0.1" height="15.0" fill="rgb(206,209,2)" rx="2" ry="2" />
<text  x="232.17" y="815.5" ></text>
</g>
<g >
<title>runtime.mallocgc (191 samples, 0.06%)</title><rect x="929.6" y="789" width="0.7" height="15.0" fill="rgb(226,179,53)" rx="2" ry="2" />
<text  x="932.64" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (357 samples, 0.11%)</title><rect x="1065.2" y="677" width="1.3" height="15.0" fill="rgb(242,151,21)" rx="2" ry="2" />
<text  x="1068.23" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*EventPoll).ReadEvents (1,511 samples, 0.46%)</title><rect x="163.5" y="741" width="5.5" height="15.0" fill="rgb(219,116,11)" rx="2" ry="2" />
<text  x="166.50" y="751.5" ></text>
</g>
<g >
<title>runtime.runOneTimer (206 samples, 0.06%)</title><rect x="113.7" y="597" width="0.7" height="15.0" fill="rgb(252,215,32)" rx="2" ry="2" />
<text  x="116.70" y="607.5" ></text>
</g>
<g >
<title>runtime.stopm (131 samples, 0.04%)</title><rect x="1007.2" y="741" width="0.5" height="15.0" fill="rgb(216,17,25)" rx="2" ry="2" />
<text  x="1010.19" y="751.5" ></text>
</g>
<g >
<title>runtime.ready (722 samples, 0.22%)</title><rect x="210.5" y="357" width="2.6" height="15.0" fill="rgb(213,5,21)" rx="2" ry="2" />
<text  x="213.53" y="367.5" ></text>
</g>
<g >
<title>runtime.exitsyscall (813 samples, 0.25%)</title><rect x="842.1" y="805" width="2.9" height="15.0" fill="rgb(243,0,16)" rx="2" ry="2" />
<text  x="845.06" y="815.5" ></text>
</g>
<g >
<title>runtime.markroot (6,110 samples, 1.87%)</title><rect x="1132.8" y="821" width="22.1" height="15.0" fill="rgb(242,109,45)" rx="2" ry="2" />
<text  x="1135.77" y="831.5" >r..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (95 samples, 0.03%)</title><rect x="13.6" y="693" width="0.4" height="15.0" fill="rgb(228,51,4)" rx="2" ry="2" />
<text  x="16.63" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (297 samples, 0.09%)</title><rect x="1186.5" y="741" width="1.1" height="15.0" fill="rgb(219,76,35)" rx="2" ry="2" />
<text  x="1189.54" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (93 samples, 0.03%)</title><rect x="904.6" y="629" width="0.3" height="15.0" fill="rgb(218,192,38)" rx="2" ry="2" />
<text  x="907.57" y="639.5" ></text>
</g>
<g >
<title>runtime.gcDrainN (49 samples, 0.02%)</title><rect x="1106.6" y="773" width="0.2" height="15.0" fill="rgb(230,185,36)" rx="2" ry="2" />
<text  x="1109.62" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (153 samples, 0.05%)</title><rect x="1128.0" y="645" width="0.6" height="15.0" fill="rgb(232,64,45)" rx="2" ry="2" />
<text  x="1131.01" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (49 samples, 0.02%)</title><rect x="776.5" y="677" width="0.1" height="15.0" fill="rgb(210,192,42)" rx="2" ry="2" />
<text  x="779.46" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (72 samples, 0.02%)</title><rect x="452.2" y="709" width="0.3" height="15.0" fill="rgb(252,79,42)" rx="2" ry="2" />
<text  x="455.24" y="719.5" ></text>
</g>
<g >
<title>runtime.wakep (130 samples, 0.04%)</title><rect x="1028.3" y="501" width="0.5" height="15.0" fill="rgb(206,159,42)" rx="2" ry="2" />
<text  x="1031.31" y="511.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).getOrCreateEntry (62 samples, 0.02%)</title><rect x="1029.3" y="613" width="0.3" height="15.0" fill="rgb(251,22,1)" rx="2" ry="2" />
<text  x="1032.34" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/packetsocket.(*endpoint).WritePacket (92 samples, 0.03%)</title><rect x="181.1" y="437" width="0.3" height="15.0" fill="rgb(225,66,25)" rx="2" ry="2" />
<text  x="184.07" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/auth.CredentialsFromContext (29 samples, 0.01%)</title><rect x="188.7" y="613" width="0.1" height="15.0" fill="rgb(248,186,13)" rx="2" ry="2" />
<text  x="191.69" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (49 samples, 0.02%)</title><rect x="18.5" y="613" width="0.2" height="15.0" fill="rgb(206,34,10)" rx="2" ry="2" />
<text  x="21.49" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (324 samples, 0.10%)</title><rect x="211.9" y="197" width="1.2" height="15.0" fill="rgb(217,175,32)" rx="2" ry="2" />
<text  x="214.94" y="207.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (78 samples, 0.02%)</title><rect x="702.6" y="613" width="0.3" height="15.0" fill="rgb(226,227,49)" rx="2" ry="2" />
<text  x="705.57" y="623.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (34 samples, 0.01%)</title><rect x="1076.5" y="677" width="0.2" height="15.0" fill="rgb(237,25,29)" rx="2" ry="2" />
<text  x="1079.54" y="687.5" ></text>
</g>
<g >
<title>runtime.chansend (76 samples, 0.02%)</title><rect x="1071.6" y="677" width="0.2" height="15.0" fill="rgb(232,19,29)" rx="2" ry="2" />
<text  x="1074.56" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (27,160 samples, 8.32%)</title><rect x="539.8" y="757" width="98.2" height="15.0" fill="rgb(218,117,39)" rx="2" ry="2" />
<text  x="542.80" y="767.5" >[[kernel.ka..</text>
</g>
<g >
<title>time.(*Timer).Reset (38 samples, 0.01%)</title><rect x="162.5" y="693" width="0.2" height="15.0" fill="rgb(228,127,51)" rx="2" ry="2" />
<text  x="165.51" y="703.5" ></text>
</g>
<g >
<title>runtime.selectgo (1,356 samples, 0.42%)</title><rect x="1185.1" y="885" width="4.9" height="15.0" fill="rgb(235,146,21)" rx="2" ry="2" />
<text  x="1188.10" y="895.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*segmentQueue).enqueue (256 samples, 0.08%)</title><rect x="759.5" y="613" width="0.9" height="15.0" fill="rgb(231,189,20)" rx="2" ry="2" />
<text  x="762.46" y="623.5" ></text>
</g>
<g >
<title>runtime.(*pageAlloc).free (109 samples, 0.03%)</title><rect x="1110.5" y="773" width="0.4" height="15.0" fill="rgb(219,221,46)" rx="2" ry="2" />
<text  x="1113.55" y="783.5" ></text>
</g>
<g >
<title>runtime.runOneTimer (71 samples, 0.02%)</title><rect x="1118.4" y="773" width="0.3" height="15.0" fill="rgb(207,58,34)" rx="2" ry="2" />
<text  x="1121.44" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*AddressableEndpointState).decAddressRefLocked (139 samples, 0.04%)</title><rect x="726.6" y="693" width="0.5" height="15.0" fill="rgb(219,113,52)" rx="2" ry="2" />
<text  x="729.57" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress.func1 (473 samples, 0.14%)</title><rect x="784.8" y="677" width="1.7" height="15.0" fill="rgb(253,7,20)" rx="2" ry="2" />
<text  x="787.76" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (196 samples, 0.06%)</title><rect x="989.0" y="101" width="0.7" height="15.0" fill="rgb(218,190,47)" rx="2" ry="2" />
<text  x="991.97" y="111.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/marshal/primitive.CopyUint32Out (77 samples, 0.02%)</title><rect x="57.2" y="725" width="0.2" height="15.0" fill="rgb(208,107,52)" rx="2" ry="2" />
<text  x="60.15" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/epoll.(*pollEntry).Callback (45 samples, 0.01%)</title><rect x="1000.2" y="837" width="0.1" height="15.0" fill="rgb(252,170,33)" rx="2" ry="2" />
<text  x="1003.18" y="847.5" ></text>
</g>
<g >
<title>runtime.startm (90 samples, 0.03%)</title><rect x="19.7" y="629" width="0.3" height="15.0" fill="rgb(240,98,8)" rx="2" ry="2" />
<text  x="22.66" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,095 samples, 0.34%)</title><rect x="152.3" y="533" width="4.0" height="15.0" fill="rgb(234,5,49)" rx="2" ry="2" />
<text  x="155.33" y="543.5" ></text>
</g>
<g >
<title>runtime.chanrecv1 (151 samples, 0.05%)</title><rect x="1004.5" y="837" width="0.5" height="15.0" fill="rgb(244,96,42)" rx="2" ry="2" />
<text  x="1007.49" y="847.5" ></text>
</g>
<g >
<title>runtime.usleep (276 samples, 0.08%)</title><rect x="700.6" y="741" width="1.0" height="15.0" fill="rgb(253,195,14)" rx="2" ry="2" />
<text  x="703.59" y="751.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (111 samples, 0.03%)</title><rect x="230.7" y="773" width="0.4" height="15.0" fill="rgb(234,119,40)" rx="2" ry="2" />
<text  x="233.66" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).Broadcast (139 samples, 0.04%)</title><rect x="735.1" y="645" width="0.5" height="15.0" fill="rgb(254,177,25)" rx="2" ry="2" />
<text  x="738.10" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).existingPMAsLocked (40 samples, 0.01%)</title><rect x="206.3" y="581" width="0.1" height="15.0" fill="rgb(253,142,16)" rx="2" ry="2" />
<text  x="209.26" y="591.5" ></text>
</g>
<g >
<title>runtime.newobject (111 samples, 0.03%)</title><rect x="207.3" y="661" width="0.4" height="15.0" fill="rgb(213,147,51)" rx="2" ry="2" />
<text  x="210.28" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).DecRef (30 samples, 0.01%)</title><rect x="43.7" y="741" width="0.2" height="15.0" fill="rgb(254,49,14)" rx="2" ry="2" />
<text  x="46.75" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (90 samples, 0.03%)</title><rect x="701.7" y="725" width="0.3" height="15.0" fill="rgb(208,106,7)" rx="2" ry="2" />
<text  x="704.67" y="735.5" ></text>
</g>
<g >
<title>runtime.runqgrab (227 samples, 0.07%)</title><rect x="1061.7" y="725" width="0.8" height="15.0" fill="rgb(248,19,43)" rx="2" ry="2" />
<text  x="1064.68" y="735.5" ></text>
</g>
<g >
<title>runtime.evacuate (56 samples, 0.02%)</title><rect x="1023.9" y="741" width="0.2" height="15.0" fill="rgb(254,65,16)" rx="2" ry="2" />
<text  x="1026.93" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (189 samples, 0.06%)</title><rect x="1065.8" y="581" width="0.7" height="15.0" fill="rgb(240,115,16)" rx="2" ry="2" />
<text  x="1068.84" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls.AddEpoll (748 samples, 0.23%)</title><rect x="70.2" y="757" width="2.7" height="15.0" fill="rgb(246,157,31)" rx="2" ry="2" />
<text  x="73.17" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="65.8" y="485" width="0.2" height="15.0" fill="rgb(223,102,52)" rx="2" ry="2" />
<text  x="68.82" y="495.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (72 samples, 0.02%)</title><rect x="1060.5" y="581" width="0.3" height="15.0" fill="rgb(221,78,22)" rx="2" ry="2" />
<text  x="1063.54" y="591.5" ></text>
</g>
<g >
<title>runtime.(*mheap).alloc.func1 (153 samples, 0.05%)</title><rect x="766.5" y="501" width="0.6" height="15.0" fill="rgb(239,10,9)" rx="2" ry="2" />
<text  x="769.51" y="511.5" ></text>
</g>
<g >
<title>runtime.futex (1,968 samples, 0.60%)</title><rect x="911.6" y="709" width="7.1" height="15.0" fill="rgb(233,175,46)" rx="2" ry="2" />
<text  x="914.61" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (408 samples, 0.13%)</title><rect x="20.0" y="853" width="1.5" height="15.0" fill="rgb(206,68,40)" rx="2" ry="2" />
<text  x="23.04" y="863.5" ></text>
</g>
<g >
<title>ipv4_confirm (47 samples, 0.01%)</title><rect x="992.3" y="197" width="0.2" height="15.0" fill="rgb(218,13,53)" rx="2" ry="2" />
<text  x="995.34" y="207.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (73 samples, 0.02%)</title><rect x="1119.5" y="645" width="0.2" height="15.0" fill="rgb(244,123,9)" rx="2" ry="2" />
<text  x="1122.46" y="655.5" ></text>
</g>
<g >
<title>runtime.notewakeup (647 samples, 0.20%)</title><rect x="210.8" y="309" width="2.3" height="15.0" fill="rgb(232,130,36)" rx="2" ry="2" />
<text  x="213.79" y="319.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (410 samples, 0.13%)</title><rect x="705.2" y="709" width="1.5" height="15.0" fill="rgb(233,86,41)" rx="2" ry="2" />
<text  x="708.21" y="719.5" ></text>
</g>
<g >
<title>time.Now (66 samples, 0.02%)</title><rect x="768.4" y="613" width="0.2" height="15.0" fill="rgb(221,127,40)" rx="2" ry="2" />
<text  x="771.37" y="623.5" ></text>
</g>
<g >
<title>runtime.resetspinning (801 samples, 0.25%)</title><rect x="919.0" y="757" width="2.9" height="15.0" fill="rgb(225,140,33)" rx="2" ry="2" />
<text  x="922.01" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header/parse.IPv4 (34 samples, 0.01%)</title><rect x="722.5" y="741" width="0.1" height="15.0" fill="rgb(215,45,37)" rx="2" ry="2" />
<text  x="725.47" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).writePacket (506 samples, 0.16%)</title><rect x="1091.0" y="725" width="1.8" height="15.0" fill="rgb(206,6,30)" rx="2" ry="2" />
<text  x="1094.02" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,142 samples, 1.27%)</title><rect x="977.6" y="245" width="15.0" height="15.0" fill="rgb(244,82,2)" rx="2" ry="2" />
<text  x="980.62" y="255.5" ></text>
</g>
<g >
<title>runtime.newobject (92 samples, 0.03%)</title><rect x="183.1" y="693" width="0.3" height="15.0" fill="rgb(205,53,17)" rx="2" ry="2" />
<text  x="186.09" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/fd.(*FD).WriteAt (28 samples, 0.01%)</title><rect x="193.3" y="533" width="0.1" height="15.0" fill="rgb(238,223,52)" rx="2" ry="2" />
<text  x="196.27" y="543.5" ></text>
</g>
<g >
<title>runtime.mapassign (42 samples, 0.01%)</title><rect x="1105.2" y="805" width="0.1" height="15.0" fill="rgb(239,169,51)" rx="2" ry="2" />
<text  x="1108.18" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (98 samples, 0.03%)</title><rect x="1108.7" y="677" width="0.3" height="15.0" fill="rgb(223,60,0)" rx="2" ry="2" />
<text  x="1111.68" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.getClock (56 samples, 0.02%)</title><rect x="63.2" y="757" width="0.2" height="15.0" fill="rgb(238,177,47)" rx="2" ry="2" />
<text  x="66.20" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (35 samples, 0.01%)</title><rect x="1109.2" y="677" width="0.1" height="15.0" fill="rgb(209,201,3)" rx="2" ry="2" />
<text  x="1112.17" y="687.5" ></text>
</g>
<g >
<title>fmt.(*fmt).fmtInteger (58 samples, 0.02%)</title><rect x="48.9" y="629" width="0.2" height="15.0" fill="rgb(210,39,37)" rx="2" ry="2" />
<text  x="51.86" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborEntry).setStateLocked (2,356 samples, 0.72%)</title><rect x="1078.5" y="741" width="8.5" height="15.0" fill="rgb(238,50,50)" rx="2" ry="2" />
<text  x="1081.46" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).WritePacket (1,413 samples, 0.43%)</title><rect x="209.4" y="565" width="5.1" height="15.0" fill="rgb(228,165,30)" rx="2" ry="2" />
<text  x="212.42" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (395 samples, 0.12%)</title><rect x="920.4" y="645" width="1.5" height="15.0" fill="rgb(224,92,23)" rx="2" ry="2" />
<text  x="923.45" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (119 samples, 0.04%)</title><rect x="734.4" y="677" width="0.4" height="15.0" fill="rgb(232,15,10)" rx="2" ry="2" />
<text  x="737.42" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.newBackoffTimer (96 samples, 0.03%)</title><rect x="1019.5" y="837" width="0.3" height="15.0" fill="rgb(250,21,2)" rx="2" ry="2" />
<text  x="1022.49" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (860 samples, 0.26%)</title><rect x="198.0" y="341" width="3.1" height="15.0" fill="rgb(251,229,35)" rx="2" ry="2" />
<text  x="200.98" y="351.5" ></text>
</g>
<g >
<title>runtime.park_m (429 samples, 0.13%)</title><rect x="705.2" y="805" width="1.5" height="15.0" fill="rgb(215,74,14)" rx="2" ry="2" />
<text  x="708.15" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (30 samples, 0.01%)</title><rect x="1173.1" y="757" width="0.1" height="15.0" fill="rgb(238,44,46)" rx="2" ry="2" />
<text  x="1176.10" y="767.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (139 samples, 0.04%)</title><rect x="1042.2" y="725" width="0.5" height="15.0" fill="rgb(205,154,48)" rx="2" ry="2" />
<text  x="1045.15" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (99 samples, 0.03%)</title><rect x="921.5" y="517" width="0.4" height="15.0" fill="rgb(236,67,51)" rx="2" ry="2" />
<text  x="924.52" y="527.5" ></text>
</g>
<g >
<title>runtime.addtimer (72 samples, 0.02%)</title><rect x="1069.1" y="805" width="0.2" height="15.0" fill="rgb(215,202,42)" rx="2" ry="2" />
<text  x="1072.05" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).IsLoopback (65 samples, 0.02%)</title><rect x="1035.2" y="709" width="0.2" height="15.0" fill="rgb(221,228,19)" rx="2" ry="2" />
<text  x="1038.18" y="719.5" ></text>
</g>
<g >
<title>runtime.scanblock (57 samples, 0.02%)</title><rect x="1147.1" y="773" width="0.2" height="15.0" fill="rgb(244,40,34)" rx="2" ry="2" />
<text  x="1150.05" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Write.func1.1 (421 samples, 0.13%)</title><rect x="205.5" y="677" width="1.5" height="15.0" fill="rgb(215,79,28)" rx="2" ry="2" />
<text  x="208.51" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (73 samples, 0.02%)</title><rect x="1049.5" y="501" width="0.2" height="15.0" fill="rgb(220,50,51)" rx="2" ry="2" />
<text  x="1052.48" y="511.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (64 samples, 0.02%)</title><rect x="1028.5" y="405" width="0.3" height="15.0" fill="rgb(253,87,19)" rx="2" ry="2" />
<text  x="1031.54" y="415.5" ></text>
</g>
<g >
<title>runtime.findrunnable (30 samples, 0.01%)</title><rect x="16.5" y="789" width="0.1" height="15.0" fill="rgb(254,82,27)" rx="2" ry="2" />
<text  x="19.50" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (31 samples, 0.01%)</title><rect x="1051.2" y="565" width="0.2" height="15.0" fill="rgb(231,117,12)" rx="2" ry="2" />
<text  x="1054.24" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doStop (99 samples, 0.03%)</title><rect x="34.3" y="853" width="0.3" height="15.0" fill="rgb(248,75,43)" rx="2" ry="2" />
<text  x="37.29" y="863.5" ></text>
</g>
<g >
<title>runtime.startm (745 samples, 0.23%)</title><rect x="919.2" y="725" width="2.7" height="15.0" fill="rgb(254,86,23)" rx="2" ry="2" />
<text  x="922.21" y="735.5" ></text>
</g>
<g >
<title>runtime.mstart1 (878 samples, 0.27%)</title><rect x="12.7" y="789" width="3.2" height="15.0" fill="rgb(210,59,4)" rx="2" ry="2" />
<text  x="15.72" y="799.5" ></text>
</g>
<g >
<title>runtime.memmove (128 samples, 0.04%)</title><rect x="1154.1" y="757" width="0.4" height="15.0" fill="rgb(211,38,15)" rx="2" ry="2" />
<text  x="1157.08" y="767.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (99 samples, 0.03%)</title><rect x="50.7" y="629" width="0.3" height="15.0" fill="rgb(253,101,5)" rx="2" ry="2" />
<text  x="53.66" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (46 samples, 0.01%)</title><rect x="962.7" y="421" width="0.2" height="15.0" fill="rgb(240,52,2)" rx="2" ry="2" />
<text  x="965.72" y="431.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (50 samples, 0.02%)</title><rect x="13.8" y="597" width="0.2" height="15.0" fill="rgb(240,10,41)" rx="2" ry="2" />
<text  x="16.80" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (514 samples, 0.16%)</title><rect x="908.9" y="597" width="1.9" height="15.0" fill="rgb(206,151,30)" rx="2" ry="2" />
<text  x="911.92" y="607.5" ></text>
</g>
<g >
<title>runtime.adjustframe (364 samples, 0.11%)</title><rect x="1102.5" y="709" width="1.3" height="15.0" fill="rgb(246,186,40)" rx="2" ry="2" />
<text  x="1105.47" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/interrupt.(*Forwarder).Enable (164 samples, 0.05%)</title><rect x="228.4" y="805" width="0.6" height="15.0" fill="rgb(220,58,44)" rx="2" ry="2" />
<text  x="231.44" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (30 samples, 0.01%)</title><rect x="53.7" y="709" width="0.1" height="15.0" fill="rgb(217,171,26)" rx="2" ry="2" />
<text  x="56.71" y="719.5" ></text>
</g>
<g >
<title>runtime.execute (66 samples, 0.02%)</title><rect x="1113.9" y="821" width="0.3" height="15.0" fill="rgb(221,198,47)" rx="2" ry="2" />
<text  x="1116.93" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (121 samples, 0.04%)</title><rect x="1126.4" y="581" width="0.4" height="15.0" fill="rgb(247,110,15)" rx="2" ry="2" />
<text  x="1129.36" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.sendTCP (1,227 samples, 0.38%)</title><rect x="1072.3" y="725" width="4.5" height="15.0" fill="rgb(245,140,23)" rx="2" ry="2" />
<text  x="1075.33" y="735.5" ></text>
</g>
<g >
<title>runtime.runqgrab (90 samples, 0.03%)</title><rect x="22.3" y="789" width="0.3" height="15.0" fill="rgb(251,130,22)" rx="2" ry="2" />
<text  x="25.28" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).Writev (3,323 samples, 1.02%)</title><rect x="204.8" y="741" width="12.0" height="15.0" fill="rgb(213,107,48)" rx="2" ry="2" />
<text  x="207.80" y="751.5" ></text>
</g>
<g >
<title>runtime.notesleep (2,047 samples, 0.63%)</title><rect x="911.3" y="725" width="7.4" height="15.0" fill="rgb(215,31,13)" rx="2" ry="2" />
<text  x="914.34" y="735.5" ></text>
</g>
<g >
<title>runtime.ready (33 samples, 0.01%)</title><rect x="1091.7" y="533" width="0.1" height="15.0" fill="rgb(242,159,16)" rx="2" ry="2" />
<text  x="1094.70" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportEndpoints).findEndpointLocked (1,012 samples, 0.31%)</title><rect x="769.7" y="677" width="3.7" height="15.0" fill="rgb(254,99,34)" rx="2" ry="2" />
<text  x="772.75" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).MaxHeaderLength (45 samples, 0.01%)</title><rect x="1072.9" y="677" width="0.1" height="15.0" fill="rgb(234,114,12)" rx="2" ry="2" />
<text  x="1075.88" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (85 samples, 0.03%)</title><rect x="115.6" y="453" width="0.3" height="15.0" fill="rgb(252,43,29)" rx="2" ry="2" />
<text  x="118.57" y="463.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (127 samples, 0.04%)</title><rect x="1042.2" y="709" width="0.4" height="15.0" fill="rgb(230,22,37)" rx="2" ry="2" />
<text  x="1045.15" y="719.5" ></text>
</g>
<g >
<title>runtime.lock2 (33 samples, 0.01%)</title><rect x="91.6" y="693" width="0.1" height="15.0" fill="rgb(215,199,22)" rx="2" ry="2" />
<text  x="94.56" y="703.5" ></text>
</g>
<g >
<title>runtime.sellock (102 samples, 0.03%)</title><rect x="703.2" y="837" width="0.3" height="15.0" fill="rgb(250,15,3)" rx="2" ry="2" />
<text  x="706.17" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="701.4" y="565" width="0.2" height="15.0" fill="rgb(217,41,50)" rx="2" ry="2" />
<text  x="704.43" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (44 samples, 0.01%)</title><rect x="1005.9" y="661" width="0.2" height="15.0" fill="rgb(225,183,7)" rx="2" ry="2" />
<text  x="1008.93" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).updateSndBufferUsage (54 samples, 0.02%)</title><rect x="1087.4" y="789" width="0.2" height="15.0" fill="rgb(252,151,47)" rx="2" ry="2" />
<text  x="1090.44" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).getOrCreateEntry (28 samples, 0.01%)</title><rect x="181.7" y="437" width="0.1" height="15.0" fill="rgb(210,204,45)" rx="2" ry="2" />
<text  x="184.67" y="447.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1019.0" y="741" width="0.1" height="15.0" fill="rgb(206,41,9)" rx="2" ry="2" />
<text  x="1022.02" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (86 samples, 0.03%)</title><rect x="162.1" y="709" width="0.4" height="15.0" fill="rgb(249,5,47)" rx="2" ry="2" />
<text  x="165.14" y="719.5" ></text>
</g>
<g >
<title>runtime.mallocgc (293 samples, 0.09%)</title><rect x="1012.2" y="741" width="1.1" height="15.0" fill="rgb(227,209,37)" rx="2" ry="2" />
<text  x="1015.23" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*runApp).execute (57 samples, 0.02%)</title><rect x="697.0" y="869" width="0.2" height="15.0" fill="rgb(227,16,32)" rx="2" ry="2" />
<text  x="699.96" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (53 samples, 0.02%)</title><rect x="1007.5" y="565" width="0.2" height="15.0" fill="rgb(228,34,37)" rx="2" ry="2" />
<text  x="1010.47" y="575.5" ></text>
</g>
<g >
<title>runtime.selectnbrecv (72 samples, 0.02%)</title><rect x="54.7" y="709" width="0.2" height="15.0" fill="rgb(214,198,42)" rx="2" ry="2" />
<text  x="57.67" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (748 samples, 0.23%)</title><rect x="153.6" y="453" width="2.7" height="15.0" fill="rgb(238,103,7)" rx="2" ry="2" />
<text  x="156.59" y="463.5" ></text>
</g>
<g >
<title>runtime.findfunc (40 samples, 0.01%)</title><rect x="1097.0" y="725" width="0.2" height="15.0" fill="rgb(231,211,25)" rx="2" ry="2" />
<text  x="1100.05" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).Capabilities (48 samples, 0.01%)</title><rect x="723.4" y="725" width="0.1" height="15.0" fill="rgb(213,120,2)" rx="2" ry="2" />
<text  x="726.36" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).check (224 samples, 0.07%)</title><rect x="188.1" y="661" width="0.8" height="15.0" fill="rgb(220,217,24)" rx="2" ry="2" />
<text  x="191.09" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.ParseTCPOptions (179 samples, 0.05%)</title><rect x="760.7" y="613" width="0.7" height="15.0" fill="rgb(247,40,19)" rx="2" ry="2" />
<text  x="763.73" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (51 samples, 0.02%)</title><rect x="23.5" y="709" width="0.2" height="15.0" fill="rgb(226,50,24)" rx="2" ry="2" />
<text  x="26.52" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).BlockWithDeadline (23,222 samples, 7.12%)</title><rect x="79.1" y="741" width="84.0" height="15.0" fill="rgb(211,67,10)" rx="2" ry="2" />
<text  x="82.15" y="751.5" >gvisor.de..</text>
</g>
<g >
<title>runtime.heapBitsSetType (33 samples, 0.01%)</title><rect x="51.6" y="645" width="0.2" height="15.0" fill="rgb(216,212,44)" rx="2" ry="2" />
<text  x="54.64" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isValidForOutgoing (79 samples, 0.02%)</title><rect x="214.6" y="565" width="0.3" height="15.0" fill="rgb(235,64,40)" rx="2" ry="2" />
<text  x="217.59" y="575.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (37 samples, 0.01%)</title><rect x="183.2" y="613" width="0.1" height="15.0" fill="rgb(211,83,25)" rx="2" ry="2" />
<text  x="186.18" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*timer).init.func1 (43 samples, 0.01%)</title><rect x="1107.3" y="869" width="0.1" height="15.0" fill="rgb(241,49,3)" rx="2" ry="2" />
<text  x="1110.28" y="879.5" ></text>
</g>
<g >
<title>runtime.systemstack (35 samples, 0.01%)</title><rect x="768.1" y="581" width="0.1" height="15.0" fill="rgb(230,105,8)" rx="2" ry="2" />
<text  x="771.11" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleSegments (8,090 samples, 2.48%)</title><rect x="1070.2" y="837" width="29.2" height="15.0" fill="rgb(233,145,35)" rx="2" ry="2" />
<text  x="1073.18" y="847.5" >gv..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,238 samples, 0.38%)</title><rect x="986.6" y="181" width="4.5" height="15.0" fill="rgb(248,182,30)" rx="2" ry="2" />
<text  x="989.59" y="191.5" ></text>
</g>
<g >
<title>br_handle_frame_finish (8,122 samples, 2.49%)</title><rect x="965.8" y="453" width="29.3" height="15.0" fill="rgb(231,204,2)" rx="2" ry="2" />
<text  x="968.77" y="463.5" >br..</text>
</g>
<g >
<title>runtime.shrinkstack (2,005 samples, 0.61%)</title><rect x="1147.3" y="773" width="7.3" height="15.0" fill="rgb(230,220,19)" rx="2" ry="2" />
<text  x="1150.30" y="783.5" ></text>
</g>
<g >
<title>runtime.mallocgc (137 samples, 0.04%)</title><rect x="785.4" y="613" width="0.5" height="15.0" fill="rgb(220,186,30)" rx="2" ry="2" />
<text  x="788.42" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,215 samples, 2.21%)</title><rect x="968.5" y="341" width="26.1" height="15.0" fill="rgb(246,74,15)" rx="2" ry="2" />
<text  x="971.49" y="351.5" >[..</text>
</g>
<g >
<title>runtime.notewakeup (351 samples, 0.11%)</title><rect x="1127.3" y="773" width="1.3" height="15.0" fill="rgb(213,152,42)" rx="2" ry="2" />
<text  x="1130.30" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (160 samples, 0.05%)</title><rect x="1152.5" y="693" width="0.6" height="15.0" fill="rgb(214,129,37)" rx="2" ry="2" />
<text  x="1155.53" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*packetEndpointList).forEach (56 samples, 0.02%)</title><rect x="795.4" y="757" width="0.2" height="15.0" fill="rgb(245,188,53)" rx="2" ry="2" />
<text  x="798.36" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (134 samples, 0.04%)</title><rect x="212.6" y="133" width="0.5" height="15.0" fill="rgb(253,38,32)" rx="2" ry="2" />
<text  x="215.63" y="143.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.parseSynSegmentOptions (41 samples, 0.01%)</title><rect x="1044.4" y="837" width="0.1" height="15.0" fill="rgb(210,27,34)" rx="2" ry="2" />
<text  x="1047.38" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,406 samples, 0.74%)</title><rect x="813.5" y="677" width="8.7" height="15.0" fill="rgb(231,91,46)" rx="2" ry="2" />
<text  x="816.46" y="687.5" ></text>
</g>
<g >
<title>runtime.schedule (270 samples, 0.08%)</title><rect x="1006.7" y="773" width="1.0" height="15.0" fill="rgb(249,196,53)" rx="2" ry="2" />
<text  x="1009.71" y="783.5" ></text>
</g>
<g >
<title>io.ReadAtLeast (284 samples, 0.09%)</title><rect x="205.6" y="661" width="1.1" height="15.0" fill="rgb(217,140,4)" rx="2" ry="2" />
<text  x="208.65" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (280 samples, 0.09%)</title><rect x="1065.5" y="661" width="1.0" height="15.0" fill="rgb(226,138,54)" rx="2" ry="2" />
<text  x="1068.51" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).existingPMAsLocked (53 samples, 0.02%)</title><rect x="176.7" y="613" width="0.1" height="15.0" fill="rgb(250,91,45)" rx="2" ry="2" />
<text  x="179.65" y="623.5" ></text>
</g>
<g >
<title>runtime.gcBgMarkWorker (155 samples, 0.05%)</title><rect x="1111.0" y="869" width="0.6" height="15.0" fill="rgb(216,214,7)" rx="2" ry="2" />
<text  x="1114.03" y="879.5" ></text>
</g>
<g >
<title>runtime.systemstack (90 samples, 0.03%)</title><rect x="19.7" y="693" width="0.3" height="15.0" fill="rgb(246,63,10)" rx="2" ry="2" />
<text  x="22.66" y="703.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (79 samples, 0.02%)</title><rect x="1051.1" y="709" width="0.3" height="15.0" fill="rgb(205,7,17)" rx="2" ry="2" />
<text  x="1054.08" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).WritePacket (100 samples, 0.03%)</title><rect x="1074.0" y="565" width="0.4" height="15.0" fill="rgb(247,196,50)" rx="2" ry="2" />
<text  x="1076.99" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).initGSO (182 samples, 0.06%)</title><rect x="1038.1" y="789" width="0.7" height="15.0" fill="rgb(209,157,17)" rx="2" ry="2" />
<text  x="1041.13" y="799.5" ></text>
</g>
<g >
<title>runtime.mallocgc (211 samples, 0.06%)</title><rect x="1106.0" y="837" width="0.8" height="15.0" fill="rgb(212,194,12)" rx="2" ry="2" />
<text  x="1109.03" y="847.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (89 samples, 0.03%)</title><rect x="56.0" y="693" width="0.3" height="15.0" fill="rgb(233,35,14)" rx="2" ry="2" />
<text  x="59.01" y="703.5" ></text>
</g>
<g >
<title>runtime.mallocgc (92 samples, 0.03%)</title><rect x="51.4" y="661" width="0.4" height="15.0" fill="rgb(216,85,31)" rx="2" ry="2" />
<text  x="54.43" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.BlockSeq.DropFirst64 (34 samples, 0.01%)</title><rect x="74.2" y="693" width="0.1" height="15.0" fill="rgb(211,96,33)" rx="2" ry="2" />
<text  x="77.21" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (254 samples, 0.08%)</title><rect x="1065.6" y="645" width="0.9" height="15.0" fill="rgb(251,185,21)" rx="2" ry="2" />
<text  x="1068.61" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).WritePacket (807 samples, 0.25%)</title><rect x="210.4" y="437" width="2.9" height="15.0" fill="rgb(216,41,7)" rx="2" ry="2" />
<text  x="213.37" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/buffer.(*VectorisedView).TrimFront (29 samples, 0.01%)</title><rect x="845.4" y="805" width="0.1" height="15.0" fill="rgb(217,104,35)" rx="2" ry="2" />
<text  x="848.43" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportEndpoints).iterEndpointsLocked (916 samples, 0.28%)</title><rect x="770.0" y="661" width="3.3" height="15.0" fill="rgb(205,102,45)" rx="2" ry="2" />
<text  x="772.96" y="671.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (245 samples, 0.08%)</title><rect x="31.5" y="869" width="0.9" height="15.0" fill="rgb(211,15,24)" rx="2" ry="2" />
<text  x="34.52" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (162 samples, 0.05%)</title><rect x="1130.3" y="677" width="0.6" height="15.0" fill="rgb(234,95,7)" rx="2" ry="2" />
<text  x="1133.32" y="687.5" ></text>
</g>
<g >
<title>runtime.newobject (45 samples, 0.01%)</title><rect x="1040.7" y="741" width="0.2" height="15.0" fill="rgb(242,170,22)" rx="2" ry="2" />
<text  x="1043.70" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (52,559 samples, 16.11%)</title><rect x="256.6" y="693" width="190.1" height="15.0" fill="rgb(254,161,52)" rx="2" ry="2" />
<text  x="259.58" y="703.5" >[[kernel.kallsyms]]</text>
</g>
<g >
<title>ext4_inode_csum_set (61 samples, 0.02%)</title><rect x="200.5" y="213" width="0.2" height="15.0" fill="rgb(221,168,30)" rx="2" ry="2" />
<text  x="203.47" y="223.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (66 samples, 0.02%)</title><rect x="23.5" y="725" width="0.2" height="15.0" fill="rgb(227,115,36)" rx="2" ry="2" />
<text  x="26.46" y="735.5" ></text>
</g>
<g >
<title>runtime.findrunnable (671 samples, 0.21%)</title><rect x="1185.2" y="821" width="2.4" height="15.0" fill="rgb(208,165,54)" rx="2" ry="2" />
<text  x="1188.21" y="831.5" ></text>
</g>
<g >
<title>runtime.markroot.func1 (6,079 samples, 1.86%)</title><rect x="1132.8" y="805" width="22.0" height="15.0" fill="rgb(250,84,19)" rx="2" ry="2" />
<text  x="1135.78" y="815.5" >r..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).check (267 samples, 0.08%)</title><rect x="188.0" y="693" width="1.0" height="15.0" fill="rgb(205,165,6)" rx="2" ry="2" />
<text  x="190.99" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (41 samples, 0.01%)</title><rect x="1078.6" y="693" width="0.2" height="15.0" fill="rgb(223,107,25)" rx="2" ry="2" />
<text  x="1081.64" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/usermem.CopyOutVec (209 samples, 0.06%)</title><rect x="176.2" y="661" width="0.8" height="15.0" fill="rgb(254,226,22)" rx="2" ry="2" />
<text  x="179.22" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (30 samples, 0.01%)</title><rect x="726.0" y="661" width="0.1" height="15.0" fill="rgb(229,83,22)" rx="2" ry="2" />
<text  x="729.03" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).walk (499 samples, 0.15%)</title><rect x="185.7" y="693" width="1.8" height="15.0" fill="rgb(238,188,50)" rx="2" ry="2" />
<text  x="188.71" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (406 samples, 0.12%)</title><rect x="705.2" y="677" width="1.5" height="15.0" fill="rgb(222,186,40)" rx="2" ry="2" />
<text  x="708.23" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).maxOptionSize (36 samples, 0.01%)</title><rect x="1010.5" y="741" width="0.1" height="15.0" fill="rgb(238,124,46)" rx="2" ry="2" />
<text  x="1013.50" y="751.5" ></text>
</g>
<g >
<title>runtime.casgstatus (170 samples, 0.05%)</title><rect x="447.7" y="725" width="0.6" height="15.0" fill="rgb(243,221,29)" rx="2" ry="2" />
<text  x="450.71" y="735.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (55 samples, 0.02%)</title><rect x="1100.2" y="789" width="0.2" height="15.0" fill="rgb(210,120,48)" rx="2" ry="2" />
<text  x="1103.17" y="799.5" ></text>
</g>
<g >
<title>runtime.mallocgc (85 samples, 0.03%)</title><rect x="1076.5" y="693" width="0.3" height="15.0" fill="rgb(243,11,22)" rx="2" ry="2" />
<text  x="1079.46" y="703.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (83 samples, 0.03%)</title><rect x="735.3" y="629" width="0.3" height="15.0" fill="rgb(238,118,30)" rx="2" ry="2" />
<text  x="738.28" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (104 samples, 0.03%)</title><rect x="738.6" y="677" width="0.4" height="15.0" fill="rgb(228,10,2)" rx="2" ry="2" />
<text  x="741.64" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (11,401 samples, 3.49%)</title><rect x="654.4" y="789" width="41.2" height="15.0" fill="rgb(244,7,0)" rx="2" ry="2" />
<text  x="657.38" y="799.5" >[[k..</text>
</g>
<g >
<title>runtime.casgstatus (67 samples, 0.02%)</title><rect x="449.4" y="741" width="0.3" height="15.0" fill="rgb(235,145,38)" rx="2" ry="2" />
<text  x="452.44" y="751.5" ></text>
</g>
<g >
<title>runtime.exitsyscallfast_pidle (390 samples, 0.12%)</title><rect x="843.5" y="757" width="1.4" height="15.0" fill="rgb(225,136,10)" rx="2" ry="2" />
<text  x="846.46" y="767.5" ></text>
</g>
<g >
<title>runtime.mapiterinit (118 samples, 0.04%)</title><rect x="737.8" y="677" width="0.4" height="15.0" fill="rgb(247,70,46)" rx="2" ry="2" />
<text  x="740.81" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*pmaSet).Find (40 samples, 0.01%)</title><rect x="176.7" y="597" width="0.1" height="15.0" fill="rgb(216,153,17)" rx="2" ry="2" />
<text  x="179.70" y="607.5" ></text>
</g>
<g >
<title>runtime.unlock2 (28 samples, 0.01%)</title><rect x="1118.7" y="789" width="0.1" height="15.0" fill="rgb(232,2,49)" rx="2" ry="2" />
<text  x="1121.70" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).setFPRegs (32 samples, 0.01%)</title><rect x="229.3" y="805" width="0.1" height="15.0" fill="rgb(240,175,47)" rx="2" ry="2" />
<text  x="232.27" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (122 samples, 0.04%)</title><rect x="157.4" y="533" width="0.4" height="15.0" fill="rgb(225,10,47)" rx="2" ry="2" />
<text  x="160.38" y="543.5" ></text>
</g>
<g >
<title>runtime.newobject (31 samples, 0.01%)</title><rect x="49.8" y="645" width="0.2" height="15.0" fill="rgb(236,121,6)" rx="2" ry="2" />
<text  x="52.85" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (33 samples, 0.01%)</title><rect x="1150.9" y="565" width="0.1" height="15.0" fill="rgb(224,121,7)" rx="2" ry="2" />
<text  x="1153.91" y="575.5" ></text>
</g>
<g >
<title>runtime.mallocgc (232 samples, 0.07%)</title><rect x="50.4" y="661" width="0.9" height="15.0" fill="rgb(245,43,2)" rx="2" ry="2" />
<text  x="53.43" y="671.5" ></text>
</g>
<g >
<title>runtime.scanstack.func1 (2,493 samples, 0.76%)</title><rect x="1137.5" y="757" width="9.0" height="15.0" fill="rgb(243,222,15)" rx="2" ry="2" />
<text  x="1140.49" y="767.5" ></text>
</g>
<g >
<title>[[vdso]] (1,189 samples, 0.36%)</title><rect x="24.6" y="853" width="4.3" height="15.0" fill="rgb(215,214,4)" rx="2" ry="2" />
<text  x="27.64" y="863.5" ></text>
</g>
<g >
<title>runtime.(*pallocBits).summarize (101 samples, 0.03%)</title><rect x="1110.6" y="757" width="0.3" height="15.0" fill="rgb(241,62,33)" rx="2" ry="2" />
<text  x="1113.58" y="767.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (483 samples, 0.15%)</title><rect x="765.3" y="533" width="1.8" height="15.0" fill="rgb(213,29,5)" rx="2" ry="2" />
<text  x="768.32" y="543.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (85 samples, 0.03%)</title><rect x="1062.2" y="661" width="0.3" height="15.0" fill="rgb(215,95,1)" rx="2" ry="2" />
<text  x="1065.19" y="671.5" ></text>
</g>
<g >
<title>runtime.selunlock (36 samples, 0.01%)</title><rect x="703.5" y="837" width="0.2" height="15.0" fill="rgb(226,3,50)" rx="2" ry="2" />
<text  x="706.54" y="847.5" ></text>
</g>
<g >
<title>[unknown] (6,084 samples, 1.86%)</title><rect x="10.5" y="885" width="22.0" height="15.0" fill="rgb(225,150,23)" rx="2" ry="2" />
<text  x="13.47" y="895.5" >[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (28 samples, 0.01%)</title><rect x="725.8" y="629" width="0.1" height="15.0" fill="rgb(232,169,13)" rx="2" ry="2" />
<text  x="728.78" y="639.5" ></text>
</g>
<g >
<title>runtime.runtimer (252 samples, 0.08%)</title><rect x="113.5" y="613" width="0.9" height="15.0" fill="rgb(209,208,16)" rx="2" ry="2" />
<text  x="116.54" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1083.4" y="341" width="0.1" height="15.0" fill="rgb(234,229,20)" rx="2" ry="2" />
<text  x="1086.36" y="351.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (925 samples, 0.28%)</title><rect x="152.9" y="485" width="3.4" height="15.0" fill="rgb(233,148,27)" rx="2" ry="2" />
<text  x="155.95" y="495.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="23.6" y="629" width="0.1" height="15.0" fill="rgb(236,205,9)" rx="2" ry="2" />
<text  x="26.58" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/arch.(*context64).SetReturn (128 samples, 0.04%)</title><rect x="39.5" y="805" width="0.5" height="15.0" fill="rgb(239,229,25)" rx="2" ry="2" />
<text  x="42.53" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (54 samples, 0.02%)</title><rect x="776.3" y="677" width="0.2" height="15.0" fill="rgb(230,99,26)" rx="2" ry="2" />
<text  x="779.26" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).Value (46 samples, 0.01%)</title><rect x="190.8" y="709" width="0.1" height="15.0" fill="rgb(210,64,34)" rx="2" ry="2" />
<text  x="193.76" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1185.9" y="709" width="0.1" height="15.0" fill="rgb(208,164,35)" rx="2" ry="2" />
<text  x="1188.90" y="719.5" ></text>
</g>
<g >
<title>runtime.scanstack (5,979 samples, 1.83%)</title><rect x="1133.0" y="789" width="21.6" height="15.0" fill="rgb(211,32,32)" rx="2" ry="2" />
<text  x="1135.95" y="799.5" >r..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (55 samples, 0.02%)</title><rect x="1151.0" y="693" width="0.2" height="15.0" fill="rgb(212,36,21)" rx="2" ry="2" />
<text  x="1154.03" y="703.5" ></text>
</g>
<g >
<title>runtime.notesleep (33 samples, 0.01%)</title><rect x="1129.5" y="773" width="0.2" height="15.0" fill="rgb(228,161,17)" rx="2" ry="2" />
<text  x="1132.55" y="783.5" ></text>
</g>
<g >
<title>runtime.futex (105 samples, 0.03%)</title><rect x="23.3" y="757" width="0.4" height="15.0" fill="rgb(237,182,23)" rx="2" ry="2" />
<text  x="26.32" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).existingPMAsLocked (165 samples, 0.05%)</title><rect x="75.5" y="693" width="0.6" height="15.0" fill="rgb(209,161,1)" rx="2" ry="2" />
<text  x="78.54" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.AddressMask.Prefix (37 samples, 0.01%)</title><rect x="735.6" y="661" width="0.1" height="15.0" fill="rgb(239,2,35)" rx="2" ry="2" />
<text  x="738.62" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*handleReadWriter).WriteFromBlocks (2,507 samples, 0.77%)</title><rect x="193.1" y="581" width="9.0" height="15.0" fill="rgb(242,1,5)" rx="2" ry="2" />
<text  x="196.08" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="1150.9" y="645" width="0.1" height="15.0" fill="rgb(253,94,31)" rx="2" ry="2" />
<text  x="1153.87" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.timeStampOffset (87 samples, 0.03%)</title><rect x="1041.2" y="773" width="0.3" height="15.0" fill="rgb(210,155,4)" rx="2" ry="2" />
<text  x="1044.18" y="783.5" ></text>
</g>
<g >
<title>runtime.getitab (32 samples, 0.01%)</title><rect x="792.5" y="709" width="0.1" height="15.0" fill="rgb(217,199,35)" rx="2" ry="2" />
<text  x="795.48" y="719.5" ></text>
</g>
<g >
<title>runtime.mallocgc (31 samples, 0.01%)</title><rect x="1081.2" y="613" width="0.1" height="15.0" fill="rgb(224,210,41)" rx="2" ry="2" />
<text  x="1084.18" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (439 samples, 0.13%)</title><rect x="154.7" y="421" width="1.6" height="15.0" fill="rgb(225,209,21)" rx="2" ry="2" />
<text  x="157.71" y="431.5" ></text>
</g>
<g >
<title>runtime.startm (201 samples, 0.06%)</title><rect x="702.1" y="757" width="0.8" height="15.0" fill="rgb(223,176,44)" rx="2" ry="2" />
<text  x="705.14" y="767.5" ></text>
</g>
<g >
<title>runtime.pcvalue (152 samples, 0.05%)</title><rect x="1097.2" y="709" width="0.6" height="15.0" fill="rgb(236,99,27)" rx="2" ry="2" />
<text  x="1100.21" y="719.5" ></text>
</g>
<g >
<title>runtime.findrunnable (34 samples, 0.01%)</title><rect x="54.5" y="645" width="0.1" height="15.0" fill="rgb(241,174,25)" rx="2" ry="2" />
<text  x="57.52" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut (68 samples, 0.02%)</title><rect x="57.2" y="677" width="0.2" height="15.0" fill="rgb(232,149,41)" rx="2" ry="2" />
<text  x="60.16" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).getOrCreateEntry (49 samples, 0.02%)</title><rect x="213.7" y="453" width="0.2" height="15.0" fill="rgb(215,128,2)" rx="2" ry="2" />
<text  x="216.74" y="463.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (70 samples, 0.02%)</title><rect x="198.5" y="293" width="0.2" height="15.0" fill="rgb(206,51,39)" rx="2" ry="2" />
<text  x="201.45" y="303.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (72 samples, 0.02%)</title><rect x="452.2" y="725" width="0.3" height="15.0" fill="rgb(219,169,46)" rx="2" ry="2" />
<text  x="455.24" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).rcvWndScaleForHandshake (47 samples, 0.01%)</title><rect x="1024.4" y="757" width="0.2" height="15.0" fill="rgb(232,31,6)" rx="2" ry="2" />
<text  x="1027.42" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (89 samples, 0.03%)</title><rect x="737.1" y="645" width="0.3" height="15.0" fill="rgb(223,152,51)" rx="2" ry="2" />
<text  x="740.08" y="655.5" ></text>
</g>
<g >
<title>runtime.mallocgc (32 samples, 0.01%)</title><rect x="182.5" y="549" width="0.1" height="15.0" fill="rgb(247,142,53)" rx="2" ry="2" />
<text  x="185.46" y="559.5" ></text>
</g>
<g >
<title>runtime.newobject (77 samples, 0.02%)</title><rect x="1084.2" y="709" width="0.3" height="15.0" fill="rgb(232,55,16)" rx="2" ry="2" />
<text  x="1087.22" y="719.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (44 samples, 0.01%)</title><rect x="171.7" y="709" width="0.1" height="15.0" fill="rgb(225,55,39)" rx="2" ry="2" />
<text  x="174.67" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.PseudoHeaderChecksum (31 samples, 0.01%)</title><rect x="182.3" y="549" width="0.1" height="15.0" fill="rgb(234,121,26)" rx="2" ry="2" />
<text  x="185.28" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/marshal/primitive.(*Uint32).CopyIn (128 samples, 0.04%)</title><rect x="56.7" y="709" width="0.4" height="15.0" fill="rgb(246,106,21)" rx="2" ry="2" />
<text  x="59.66" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Job).Schedule (840 samples, 0.26%)</title><rect x="1047.4" y="789" width="3.0" height="15.0" fill="rgb(247,148,1)" rx="2" ry="2" />
<text  x="1050.38" y="799.5" ></text>
</g>
<g >
<title>runtime.runtimer (145 samples, 0.04%)</title><rect x="903.7" y="725" width="0.5" height="15.0" fill="rgb(242,169,23)" rx="2" ry="2" />
<text  x="906.71" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MTU (97 samples, 0.03%)</title><rect x="1009.2" y="757" width="0.4" height="15.0" fill="rgb(237,136,19)" rx="2" ry="2" />
<text  x="1012.23" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (48 samples, 0.01%)</title><rect x="1108.9" y="565" width="0.1" height="15.0" fill="rgb(210,16,41)" rx="2" ry="2" />
<text  x="1111.86" y="575.5" ></text>
</g>
<g >
<title>runtime.(*mheap).alloc.func1 (28 samples, 0.01%)</title><rect x="50.9" y="565" width="0.1" height="15.0" fill="rgb(211,28,9)" rx="2" ry="2" />
<text  x="53.87" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (679 samples, 0.21%)</title><rect x="987.2" y="165" width="2.5" height="15.0" fill="rgb(238,16,17)" rx="2" ry="2" />
<text  x="990.23" y="175.5" ></text>
</g>
<g >
<title>runtime.findfunc (39 samples, 0.01%)</title><rect x="1149.9" y="725" width="0.1" height="15.0" fill="rgb(207,48,53)" rx="2" ry="2" />
<text  x="1152.91" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (342 samples, 0.10%)</title><rect x="1188.5" y="693" width="1.2" height="15.0" fill="rgb(240,195,20)" rx="2" ry="2" />
<text  x="1191.47" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (98 samples, 0.03%)</title><rect x="1083.1" y="437" width="0.4" height="15.0" fill="rgb(254,3,22)" rx="2" ry="2" />
<text  x="1086.11" y="447.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendTCP (704 samples, 0.22%)</title><rect x="180.3" y="597" width="2.6" height="15.0" fill="rgb(237,60,49)" rx="2" ry="2" />
<text  x="183.34" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (90 samples, 0.03%)</title><rect x="13.1" y="629" width="0.3" height="15.0" fill="rgb(227,100,34)" rx="2" ry="2" />
<text  x="16.07" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*receiver).handleRcvdSegment (1,844 samples, 0.57%)</title><rect x="1070.6" y="805" width="6.7" height="15.0" fill="rgb(224,46,4)" rx="2" ry="2" />
<text  x="1073.64" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (116 samples, 0.04%)</title><rect x="1049.3" y="549" width="0.4" height="15.0" fill="rgb(235,107,33)" rx="2" ry="2" />
<text  x="1052.32" y="559.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc.func1 (48 samples, 0.01%)</title><rect x="929.1" y="757" width="0.2" height="15.0" fill="rgb(247,87,2)" rx="2" ry="2" />
<text  x="932.08" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (45 samples, 0.01%)</title><rect x="1078.6" y="709" width="0.2" height="15.0" fill="rgb(241,32,48)" rx="2" ry="2" />
<text  x="1081.63" y="719.5" ></text>
</g>
<g >
<title>runtime.mapaccess2 (39 samples, 0.01%)</title><rect x="1101.5" y="773" width="0.2" height="15.0" fill="rgb(221,217,35)" rx="2" ry="2" />
<text  x="1104.51" y="783.5" ></text>
</g>
<g >
<title>runtime.mallocgc (58 samples, 0.02%)</title><rect x="1083.5" y="645" width="0.2" height="15.0" fill="rgb(248,151,7)" rx="2" ry="2" />
<text  x="1086.50" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (31 samples, 0.01%)</title><rect x="1079.3" y="613" width="0.1" height="15.0" fill="rgb(220,136,20)" rx="2" ry="2" />
<text  x="1082.25" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).Enabled (68 samples, 0.02%)</title><rect x="787.4" y="661" width="0.3" height="15.0" fill="rgb(222,55,13)" rx="2" ry="2" />
<text  x="790.45" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (46 samples, 0.01%)</title><rect x="23.1" y="613" width="0.2" height="15.0" fill="rgb(237,35,20)" rx="2" ry="2" />
<text  x="26.11" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isValidForOutgoingRLocked (71 samples, 0.02%)</title><rect x="1075.6" y="677" width="0.2" height="15.0" fill="rgb(240,173,29)" rx="2" ry="2" />
<text  x="1078.57" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).ID (28 samples, 0.01%)</title><rect x="781.9" y="741" width="0.1" height="15.0" fill="rgb(206,58,17)" rx="2" ry="2" />
<text  x="784.95" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (37 samples, 0.01%)</title><rect x="187.5" y="693" width="0.2" height="15.0" fill="rgb(233,87,24)" rx="2" ry="2" />
<text  x="190.54" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (47 samples, 0.01%)</title><rect x="52.0" y="645" width="0.2" height="15.0" fill="rgb(218,88,19)" rx="2" ry="2" />
<text  x="54.99" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).DeliverNetworkPacket (52 samples, 0.02%)</title><rect x="717.6" y="821" width="0.2" height="15.0" fill="rgb(245,208,41)" rx="2" ry="2" />
<text  x="720.64" y="831.5" ></text>
</g>
<g >
<title>runtime.funcspdelta (184 samples, 0.06%)</title><rect x="1054.6" y="709" width="0.7" height="15.0" fill="rgb(208,157,24)" rx="2" ry="2" />
<text  x="1057.62" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (412 samples, 0.13%)</title><rect x="1125.3" y="645" width="1.5" height="15.0" fill="rgb(224,22,29)" rx="2" ry="2" />
<text  x="1128.30" y="655.5" ></text>
</g>
<g >
<title>runtime.mapdelete (28 samples, 0.01%)</title><rect x="1100.5" y="789" width="0.1" height="15.0" fill="rgb(215,157,23)" rx="2" ry="2" />
<text  x="1103.47" y="799.5" ></text>
</g>
<g >
<title>runtime.newobject (38 samples, 0.01%)</title><rect x="47.2" y="709" width="0.1" height="15.0" fill="rgb(205,148,2)" rx="2" ry="2" />
<text  x="50.18" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (9,913 samples, 3.04%)</title><rect x="659.8" y="773" width="35.8" height="15.0" fill="rgb(228,14,9)" rx="2" ry="2" />
<text  x="662.76" y="783.5" >[[k..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (48 samples, 0.01%)</title><rect x="23.5" y="677" width="0.2" height="15.0" fill="rgb(228,191,36)" rx="2" ry="2" />
<text  x="26.53" y="687.5" ></text>
</g>
<g >
<title>runtime.runqsteal (50 samples, 0.02%)</title><rect x="29.2" y="869" width="0.2" height="15.0" fill="rgb(220,101,32)" rx="2" ry="2" />
<text  x="32.18" y="879.5" ></text>
</g>
<g >
<title>runtime.findObject (367 samples, 0.11%)</title><rect x="1143.8" y="709" width="1.4" height="15.0" fill="rgb(208,164,44)" rx="2" ry="2" />
<text  x="1146.84" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.makeSynOptions (64 samples, 0.02%)</title><rect x="1031.4" y="773" width="0.2" height="15.0" fill="rgb(243,94,14)" rx="2" ry="2" />
<text  x="1034.38" y="783.5" ></text>
</g>
<g >
<title>runtime.(*mheap).allocSpan (144 samples, 0.04%)</title><rect x="1098.2" y="693" width="0.5" height="15.0" fill="rgb(211,133,9)" rx="2" ry="2" />
<text  x="1101.20" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborEntry).setStateLocked (2,464 samples, 0.76%)</title><rect x="1047.1" y="805" width="8.9" height="15.0" fill="rgb(230,160,41)" rx="2" ry="2" />
<text  x="1050.09" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborEntry).handleUpperLevelConfirmationLocked (2,406 samples, 0.74%)</title><rect x="1078.3" y="757" width="8.7" height="15.0" fill="rgb(241,212,4)" rx="2" ry="2" />
<text  x="1081.30" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).WritePacket (481 samples, 0.15%)</title><rect x="1073.6" y="661" width="1.7" height="15.0" fill="rgb(231,73,30)" rx="2" ry="2" />
<text  x="1076.56" y="671.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).grow (118 samples, 0.04%)</title><rect x="1042.2" y="693" width="0.4" height="15.0" fill="rgb(241,179,33)" rx="2" ry="2" />
<text  x="1045.16" y="703.5" ></text>
</g>
<g >
<title>runtime.systemstack (228 samples, 0.07%)</title><rect x="1005.3" y="837" width="0.8" height="15.0" fill="rgb(205,111,3)" rx="2" ry="2" />
<text  x="1008.27" y="847.5" ></text>
</g>
<g >
<title>runtime.notesleep (100 samples, 0.03%)</title><rect x="701.6" y="757" width="0.4" height="15.0" fill="rgb(251,160,50)" rx="2" ry="2" />
<text  x="704.64" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.Writer.WriteFromBlocks-fm (47 samples, 0.01%)</title><rect x="21.6" y="693" width="0.1" height="15.0" fill="rgb(246,205,23)" rx="2" ry="2" />
<text  x="24.57" y="703.5" ></text>
</g>
<g >
<title>runtime.ready (36 samples, 0.01%)</title><rect x="1069.5" y="773" width="0.1" height="15.0" fill="rgb(207,107,53)" rx="2" ry="2" />
<text  x="1072.48" y="783.5" ></text>
</g>
<g >
<title>runtime.step (35 samples, 0.01%)</title><rect x="1098.9" y="741" width="0.1" height="15.0" fill="rgb(234,128,7)" rx="2" ry="2" />
<text  x="1101.92" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (39 samples, 0.01%)</title><rect x="69.8" y="677" width="0.2" height="15.0" fill="rgb(224,37,5)" rx="2" ry="2" />
<text  x="72.82" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (45 samples, 0.01%)</title><rect x="13.8" y="581" width="0.2" height="15.0" fill="rgb(223,70,26)" rx="2" ry="2" />
<text  x="16.82" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).isValidForOutgoing (83 samples, 0.03%)</title><rect x="1093.1" y="741" width="0.3" height="15.0" fill="rgb(237,67,8)" rx="2" ry="2" />
<text  x="1096.08" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (15,979 samples, 4.90%)</title><rect x="940.3" y="677" width="57.8" height="15.0" fill="rgb(208,115,23)" rx="2" ry="2" />
<text  x="943.34" y="687.5" >[[kern..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).getFPRegs (39 samples, 0.01%)</title><rect x="229.0" y="805" width="0.2" height="15.0" fill="rgb(250,212,10)" rx="2" ry="2" />
<text  x="232.03" y="815.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast32 (39 samples, 0.01%)</title><rect x="775.3" y="693" width="0.1" height="15.0" fill="rgb(236,128,29)" rx="2" ry="2" />
<text  x="778.30" y="703.5" ></text>
</g>
<g >
<title>runtime.step (81 samples, 0.02%)</title><rect x="1054.9" y="677" width="0.3" height="15.0" fill="rgb(209,177,20)" rx="2" ry="2" />
<text  x="1057.95" y="687.5" ></text>
</g>
<g >
<title>runtime.notesleep (126 samples, 0.04%)</title><rect x="157.4" y="613" width="0.4" height="15.0" fill="rgb(231,14,51)" rx="2" ry="2" />
<text  x="160.36" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).UnlockUser (29 samples, 0.01%)</title><rect x="177.4" y="677" width="0.1" height="15.0" fill="rgb(228,114,25)" rx="2" ry="2" />
<text  x="180.42" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (56 samples, 0.02%)</title><rect x="452.3" y="613" width="0.2" height="15.0" fill="rgb(214,172,14)" rx="2" ry="2" />
<text  x="455.30" y="623.5" ></text>
</g>
<g >
<title>runtime.mapassign (112 samples, 0.03%)</title><rect x="65.1" y="645" width="0.4" height="15.0" fill="rgb(212,152,12)" rx="2" ry="2" />
<text  x="68.08" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).assertTaskGoroutine (37 samples, 0.01%)</title><rect x="84.0" y="693" width="0.1" height="15.0" fill="rgb(243,5,29)" rx="2" ry="2" />
<text  x="86.97" y="703.5" ></text>
</g>
<g >
<title>time.(*Timer).Stop (33 samples, 0.01%)</title><rect x="1068.8" y="837" width="0.2" height="15.0" fill="rgb(230,90,28)" rx="2" ry="2" />
<text  x="1071.84" y="847.5" ></text>
</g>
<g >
<title>runtime.schedule (1,149 samples, 0.35%)</title><rect x="698.8" y="805" width="4.2" height="15.0" fill="rgb(229,127,48)" rx="2" ry="2" />
<text  x="701.83" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).protocolMainLoop.func6 (8,152 samples, 2.50%)</title><rect x="1070.1" y="853" width="29.5" height="15.0" fill="rgb(223,154,41)" rx="2" ry="2" />
<text  x="1073.13" y="863.5" >gv..</text>
</g>
<g >
<title>runtime.(*mcache).nextFree (145 samples, 0.04%)</title><rect x="1042.1" y="741" width="0.6" height="15.0" fill="rgb(216,58,53)" rx="2" ry="2" />
<text  x="1045.14" y="751.5" ></text>
</g>
<g >
<title>runtime.mallocgc (33 samples, 0.01%)</title><rect x="191.6" y="693" width="0.1" height="15.0" fill="rgb(211,108,37)" rx="2" ry="2" />
<text  x="194.62" y="703.5" ></text>
</g>
<g >
<title>runtime.copystack (790 samples, 0.24%)</title><rect x="1102.0" y="741" width="2.8" height="15.0" fill="rgb(251,193,24)" rx="2" ry="2" />
<text  x="1104.96" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (70 samples, 0.02%)</title><rect x="1153.3" y="677" width="0.2" height="15.0" fill="rgb(251,23,9)" rx="2" ry="2" />
<text  x="1156.26" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="1028.7" y="309" width="0.1" height="15.0" fill="rgb(233,172,3)" rx="2" ry="2" />
<text  x="1031.66" y="319.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).receiveBufferSize (118 samples, 0.04%)</title><rect x="759.8" y="597" width="0.4" height="15.0" fill="rgb(213,54,48)" rx="2" ry="2" />
<text  x="762.81" y="607.5" ></text>
</g>
<g >
<title>runtime.systemstack (50 samples, 0.02%)</title><rect x="1106.6" y="821" width="0.2" height="15.0" fill="rgb(211,10,29)" rx="2" ry="2" />
<text  x="1109.61" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).sendTCP (1,371 samples, 0.42%)</title><rect x="1026.4" y="773" width="5.0" height="15.0" fill="rgb(250,18,33)" rx="2" ry="2" />
<text  x="1029.42" y="783.5" ></text>
</g>
<g >
<title>runtime.mapaccess2_fast32 (180 samples, 0.06%)</title><rect x="769.0" y="661" width="0.6" height="15.0" fill="rgb(254,141,6)" rx="2" ry="2" />
<text  x="771.99" y="671.5" ></text>
</g>
<g >
<title>br_allowed_ingress (36 samples, 0.01%)</title><rect x="966.2" y="437" width="0.1" height="15.0" fill="rgb(226,3,52)" rx="2" ry="2" />
<text  x="969.20" y="447.5" ></text>
</g>
<g >
<title>runtime.greyobject (369 samples, 0.11%)</title><rect x="1145.2" y="709" width="1.3" height="15.0" fill="rgb(228,12,45)" rx="2" ry="2" />
<text  x="1148.17" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,112 samples, 2.18%)</title><rect x="968.9" y="325" width="25.7" height="15.0" fill="rgb(205,50,15)" rx="2" ry="2" />
<text  x="971.86" y="335.5" >[..</text>
</g>
<g >
<title>runtime.runqsteal (347 samples, 0.11%)</title><rect x="700.3" y="773" width="1.3" height="15.0" fill="rgb(241,34,10)" rx="2" ry="2" />
<text  x="703.35" y="783.5" ></text>
</g>
<g >
<title>runtime.notesleep (603 samples, 0.18%)</title><rect x="1062.6" y="725" width="2.2" height="15.0" fill="rgb(231,83,54)" rx="2" ry="2" />
<text  x="1065.63" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (42 samples, 0.01%)</title><rect x="1153.4" y="613" width="0.1" height="15.0" fill="rgb(239,123,51)" rx="2" ry="2" />
<text  x="1156.36" y="623.5" ></text>
</g>
<g >
<title>runtime.gentraceback (681 samples, 0.21%)</title><rect x="1102.1" y="725" width="2.4" height="15.0" fill="rgb(217,23,10)" rx="2" ry="2" />
<text  x="1105.06" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn (166 samples, 0.05%)</title><rect x="205.9" y="613" width="0.6" height="15.0" fill="rgb(213,48,49)" rx="2" ry="2" />
<text  x="208.88" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (46 samples, 0.01%)</title><rect x="204.0" y="661" width="0.2" height="15.0" fill="rgb(215,186,37)" rx="2" ry="2" />
<text  x="206.99" y="671.5" ></text>
</g>
<g >
<title>runtime.mallocgc (45 samples, 0.01%)</title><rect x="1084.3" y="693" width="0.2" height="15.0" fill="rgb(224,118,29)" rx="2" ry="2" />
<text  x="1087.29" y="703.5" ></text>
</g>
<g >
<title>nf_nat_ipv4_in (175 samples, 0.05%)</title><rect x="964.5" y="469" width="0.6" height="15.0" fill="rgb(212,224,29)" rx="2" ry="2" />
<text  x="967.49" y="479.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (144 samples, 0.04%)</title><rect x="1130.4" y="661" width="0.5" height="15.0" fill="rgb(226,136,44)" rx="2" ry="2" />
<text  x="1133.38" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).executeSyscall (40 samples, 0.01%)</title><rect x="220.2" y="805" width="0.1" height="15.0" fill="rgb(231,189,2)" rx="2" ry="2" />
<text  x="223.16" y="815.5" ></text>
</g>
<g >
<title>runtime.newstack (1,025 samples, 0.31%)</title><rect x="1052.0" y="757" width="3.7" height="15.0" fill="rgb(209,114,54)" rx="2" ry="2" />
<text  x="1054.96" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,563 samples, 0.48%)</title><rect x="195.6" y="389" width="5.6" height="15.0" fill="rgb(235,175,19)" rx="2" ry="2" />
<text  x="198.57" y="399.5" ></text>
</g>
<g >
<title>runtime.mallocgc (45 samples, 0.01%)</title><rect x="1055.8" y="773" width="0.1" height="15.0" fill="rgb(239,91,6)" rx="2" ry="2" />
<text  x="1058.77" y="783.5" ></text>
</g>
<g >
<title>br_netif_receive_skb (42 samples, 0.01%)</title><rect x="994.7" y="421" width="0.1" height="15.0" fill="rgb(236,185,2)" rx="2" ry="2" />
<text  x="997.65" y="431.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (32 samples, 0.01%)</title><rect x="1006.0" y="629" width="0.1" height="15.0" fill="rgb(236,188,5)" rx="2" ry="2" />
<text  x="1008.97" y="639.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,002 samples, 0.31%)</title><rect x="915.1" y="597" width="3.6" height="15.0" fill="rgb(209,224,28)" rx="2" ry="2" />
<text  x="918.10" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Dirent).DecRef (250 samples, 0.08%)</title><rect x="63.8" y="709" width="0.9" height="15.0" fill="rgb(248,126,20)" rx="2" ry="2" />
<text  x="66.83" y="719.5" ></text>
</g>
<g >
<title>runtime.execute (192 samples, 0.06%)</title><rect x="97.4" y="645" width="0.7" height="15.0" fill="rgb(253,43,41)" rx="2" ry="2" />
<text  x="100.42" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Accept (408 samples, 0.13%)</title><rect x="53.5" y="725" width="1.5" height="15.0" fill="rgb(220,57,10)" rx="2" ry="2" />
<text  x="56.52" y="735.5" ></text>
</g>
<g >
<title>runtime.goready.func1 (38 samples, 0.01%)</title><rect x="1069.5" y="789" width="0.1" height="15.0" fill="rgb(248,186,25)" rx="2" ry="2" />
<text  x="1072.48" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (47 samples, 0.01%)</title><rect x="790.6" y="677" width="0.2" height="15.0" fill="rgb(218,83,36)" rx="2" ry="2" />
<text  x="793.62" y="687.5" ></text>
</g>
<g >
<title>runtime.getStackMap (75 samples, 0.02%)</title><rect x="1184.0" y="805" width="0.3" height="15.0" fill="rgb(240,38,49)" rx="2" ry="2" />
<text  x="1186.99" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (65 samples, 0.02%)</title><rect x="13.7" y="629" width="0.3" height="15.0" fill="rgb(241,180,10)" rx="2" ry="2" />
<text  x="16.74" y="639.5" ></text>
</g>
<g >
<title>runtime.lock2 (90 samples, 0.03%)</title><rect x="703.2" y="821" width="0.3" height="15.0" fill="rgb(236,6,23)" rx="2" ry="2" />
<text  x="706.21" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,937 samples, 0.90%)</title><rect x="137.8" y="501" width="10.6" height="15.0" fill="rgb(226,70,5)" rx="2" ry="2" />
<text  x="140.80" y="511.5" ></text>
</g>
<g >
<title>runtime.write1 (195 samples, 0.06%)</title><rect x="1049.0" y="645" width="0.7" height="15.0" fill="rgb(235,80,31)" rx="2" ry="2" />
<text  x="1052.04" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/buffer.(*VectorisedView).ReadTo (290 samples, 0.09%)</title><rect x="175.9" y="693" width="1.1" height="15.0" fill="rgb(251,126,16)" rx="2" ry="2" />
<text  x="178.94" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*timekeeperClock).Now (50 samples, 0.02%)</title><rect x="42.8" y="773" width="0.2" height="15.0" fill="rgb(211,114,19)" rx="2" ry="2" />
<text  x="45.81" y="783.5" ></text>
</g>
<g >
<title>runtime.wakep (61 samples, 0.02%)</title><rect x="1051.1" y="677" width="0.3" height="15.0" fill="rgb(239,140,40)" rx="2" ry="2" />
<text  x="1054.13" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (68 samples, 0.02%)</title><rect x="783.2" y="693" width="0.2" height="15.0" fill="rgb(235,0,38)" rx="2" ry="2" />
<text  x="786.19" y="703.5" ></text>
</g>
<g >
<title>runtime.pcvalue (164 samples, 0.05%)</title><rect x="1054.6" y="693" width="0.6" height="15.0" fill="rgb(238,7,11)" rx="2" ry="2" />
<text  x="1057.65" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (46 samples, 0.01%)</title><rect x="199.0" y="245" width="0.2" height="15.0" fill="rgb(223,18,28)" rx="2" ry="2" />
<text  x="202.01" y="255.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (168 samples, 0.05%)</title><rect x="15.3" y="597" width="0.6" height="15.0" fill="rgb(235,37,36)" rx="2" ry="2" />
<text  x="18.26" y="607.5" ></text>
</g>
<g >
<title>runtime.newobject (133 samples, 0.04%)</title><rect x="1019.9" y="837" width="0.5" height="15.0" fill="rgb(215,110,50)" rx="2" ry="2" />
<text  x="1022.92" y="847.5" ></text>
</g>
<g >
<title>runtime.mallocgc (69 samples, 0.02%)</title><rect x="1033.9" y="693" width="0.3" height="15.0" fill="rgb(239,10,27)" rx="2" ry="2" />
<text  x="1036.92" y="703.5" ></text>
</g>
<g >
<title>runtime.heapBitsSetType (153 samples, 0.05%)</title><rect x="767.4" y="581" width="0.5" height="15.0" fill="rgb(217,116,53)" rx="2" ry="2" />
<text  x="770.37" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (68 samples, 0.02%)</title><rect x="1051.6" y="725" width="0.2" height="15.0" fill="rgb(229,187,2)" rx="2" ry="2" />
<text  x="1054.56" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (77 samples, 0.02%)</title><rect x="1152.8" y="629" width="0.3" height="15.0" fill="rgb(239,24,24)" rx="2" ry="2" />
<text  x="1155.83" y="639.5" ></text>
</g>
<g >
<title>runtime.lock2 (67 samples, 0.02%)</title><rect x="1060.8" y="741" width="0.2" height="15.0" fill="rgb(206,225,52)" rx="2" ry="2" />
<text  x="1063.80" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.NewWeakRef (209 samples, 0.06%)</title><rect x="45.4" y="709" width="0.7" height="15.0" fill="rgb(214,139,12)" rx="2" ry="2" />
<text  x="48.36" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (17,460 samples, 5.35%)</title><rect x="935.0" y="773" width="63.1" height="15.0" fill="rgb(250,195,2)" rx="2" ry="2" />
<text  x="937.99" y="783.5" >[[kern..</text>
</g>
<g >
<title>runtime.mapaccess1_fast32 (83 samples, 0.03%)</title><rect x="794.0" y="741" width="0.3" height="15.0" fill="rgb(237,124,16)" rx="2" ry="2" />
<text  x="797.01" y="751.5" ></text>
</g>
<g >
<title>runtime.chanrecv (143 samples, 0.04%)</title><rect x="1004.5" y="821" width="0.5" height="15.0" fill="rgb(208,117,54)" rx="2" ry="2" />
<text  x="1007.52" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).readyToRead (204 samples, 0.06%)</title><rect x="1071.2" y="773" width="0.7" height="15.0" fill="rgb(251,75,5)" rx="2" ry="2" />
<text  x="1074.18" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*fileOperations).Write (47 samples, 0.01%)</title><rect x="21.6" y="773" width="0.1" height="15.0" fill="rgb(218,193,23)" rx="2" ry="2" />
<text  x="24.57" y="783.5" ></text>
</g>
<g >
<title>runtime.mProf_Malloc (47 samples, 0.01%)</title><rect x="767.9" y="565" width="0.2" height="15.0" fill="rgb(241,125,49)" rx="2" ry="2" />
<text  x="770.93" y="575.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (17,071 samples, 5.23%)</title><rect x="936.4" y="741" width="61.7" height="15.0" fill="rgb(243,88,44)" rx="2" ry="2" />
<text  x="939.40" y="751.5" >[[kern..</text>
</g>
<g >
<title>runtime.systemstack (44 samples, 0.01%)</title><rect x="1042.4" y="677" width="0.2" height="15.0" fill="rgb(225,142,44)" rx="2" ry="2" />
<text  x="1045.43" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,062 samples, 0.33%)</title><rect x="837.4" y="565" width="3.8" height="15.0" fill="rgb(237,7,49)" rx="2" ry="2" />
<text  x="840.41" y="575.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (57 samples, 0.02%)</title><rect x="784.1" y="677" width="0.2" height="15.0" fill="rgb(227,120,16)" rx="2" ry="2" />
<text  x="787.14" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).writePacket (297 samples, 0.09%)</title><rect x="180.9" y="533" width="1.1" height="15.0" fill="rgb(213,155,54)" rx="2" ry="2" />
<text  x="183.95" y="543.5" ></text>
</g>
<g >
<title>runtime.newobject (34 samples, 0.01%)</title><rect x="205.0" y="709" width="0.1" height="15.0" fill="rgb(205,145,9)" rx="2" ry="2" />
<text  x="208.01" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/nested.(*Endpoint).Capabilities (225 samples, 0.07%)</title><rect x="722.7" y="741" width="0.8" height="15.0" fill="rgb(209,226,42)" rx="2" ry="2" />
<text  x="725.72" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportDemuxer).deliverPacket (9,207 samples, 2.82%)</title><rect x="740.6" y="693" width="33.3" height="15.0" fill="rgb(211,19,48)" rx="2" ry="2" />
<text  x="743.56" y="703.5" >gv..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (80 samples, 0.02%)</title><rect x="117.8" y="581" width="0.3" height="15.0" fill="rgb(205,119,40)" rx="2" ry="2" />
<text  x="120.80" y="591.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).IovecsIOSequence (236 samples, 0.07%)</title><rect x="203.7" y="757" width="0.9" height="15.0" fill="rgb(214,102,34)" rx="2" ry="2" />
<text  x="206.72" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyInTo (2,822 samples, 0.86%)</title><rect x="192.3" y="677" width="10.3" height="15.0" fill="rgb(231,40,53)" rx="2" ry="2" />
<text  x="195.35" y="687.5" ></text>
</g>
<g >
<title>runtime.adjusttimers (271 samples, 0.08%)</title><rect x="1057.2" y="741" width="1.0" height="15.0" fill="rgb(234,113,44)" rx="2" ry="2" />
<text  x="1060.22" y="751.5" ></text>
</g>
<g >
<title>runtime.pcvalue (165 samples, 0.05%)</title><rect x="1053.7" y="661" width="0.6" height="15.0" fill="rgb(211,88,43)" rx="2" ry="2" />
<text  x="1056.68" y="671.5" ></text>
</g>
<g >
<title>tcp_packet (442 samples, 0.14%)</title><rect x="962.9" y="453" width="1.6" height="15.0" fill="rgb(206,15,2)" rx="2" ry="2" />
<text  x="965.89" y="463.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).doSyscallInvoke (49,819 samples, 15.27%)</title><rect x="40.0" y="805" width="180.2" height="15.0" fill="rgb(232,87,37)" rx="2" ry="2" />
<text  x="42.99" y="815.5" >gvisor.dev/gvisor/pkg/s..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (4,266 samples, 1.31%)</title><rect x="879.5" y="677" width="15.4" height="15.0" fill="rgb(244,140,35)" rx="2" ry="2" />
<text  x="882.46" y="687.5" ></text>
</g>
<g >
<title>runtime.isSystemGoroutine (56 samples, 0.02%)</title><rect x="1005.5" y="789" width="0.2" height="15.0" fill="rgb(229,202,12)" rx="2" ry="2" />
<text  x="1008.50" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (69 samples, 0.02%)</title><rect x="452.3" y="693" width="0.2" height="15.0" fill="rgb(241,85,46)" rx="2" ry="2" />
<text  x="455.25" y="703.5" ></text>
</g>
<g >
<title>runtime.(*itabTableType).find (144 samples, 0.04%)</title><rect x="791.7" y="693" width="0.5" height="15.0" fill="rgb(237,9,29)" rx="2" ry="2" />
<text  x="794.68" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).RUnlock (43 samples, 0.01%)</title><rect x="718.8" y="789" width="0.2" height="15.0" fill="rgb(239,0,45)" rx="2" ry="2" />
<text  x="721.80" y="799.5" ></text>
</g>
<g >
<title>ovl_real_fdget_meta (30 samples, 0.01%)</title><rect x="201.3" y="389" width="0.1" height="15.0" fill="rgb(245,220,20)" rx="2" ry="2" />
<text  x="204.30" y="399.5" ></text>
</g>
<g >
<title>runtime.systemstack (48 samples, 0.01%)</title><rect x="929.1" y="773" width="0.2" height="15.0" fill="rgb(206,61,46)" rx="2" ry="2" />
<text  x="932.08" y="783.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (28 samples, 0.01%)</title><rect x="182.7" y="533" width="0.1" height="15.0" fill="rgb(242,107,4)" rx="2" ry="2" />
<text  x="185.69" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MaxHeaderLength (53 samples, 0.02%)</title><rect x="1090.5" y="757" width="0.2" height="15.0" fill="rgb(219,75,38)" rx="2" ry="2" />
<text  x="1093.46" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/ports.(*PortManager).releasePortLocked (144 samples, 0.04%)</title><rect x="1100.1" y="805" width="0.5" height="15.0" fill="rgb(214,55,52)" rx="2" ry="2" />
<text  x="1103.11" y="815.5" ></text>
</g>
<g >
<title>runtime.futex (157 samples, 0.05%)</title><rect x="22.7" y="773" width="0.6" height="15.0" fill="rgb(245,46,43)" rx="2" ry="2" />
<text  x="25.70" y="783.5" ></text>
</g>
<g >
<title>runtime.newobject (283 samples, 0.09%)</title><rect x="1105.8" y="853" width="1.1" height="15.0" fill="rgb(243,212,34)" rx="2" ry="2" />
<text  x="1108.85" y="863.5" ></text>
</g>
<g >
<title>runtime.mallocgc (33 samples, 0.01%)</title><rect x="1040.7" y="725" width="0.1" height="15.0" fill="rgb(228,141,21)" rx="2" ry="2" />
<text  x="1043.72" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (235 samples, 0.07%)</title><rect x="704.0" y="629" width="0.9" height="15.0" fill="rgb(216,139,2)" rx="2" ry="2" />
<text  x="707.03" y="639.5" ></text>
</g>
<g >
<title>br_pass_frame_up (7,627 samples, 2.34%)</title><rect x="967.2" y="437" width="27.6" height="15.0" fill="rgb(253,148,27)" rx="2" ry="2" />
<text  x="970.22" y="447.5" >b..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (62 samples, 0.02%)</title><rect x="18.4" y="645" width="0.3" height="15.0" fill="rgb(220,21,50)" rx="2" ry="2" />
<text  x="21.44" y="655.5" ></text>
</g>
<g >
<title>runtime.pcvalue (110 samples, 0.03%)</title><rect x="1104.1" y="693" width="0.4" height="15.0" fill="rgb(213,225,4)" rx="2" ry="2" />
<text  x="1107.07" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).writePacket (1,285 samples, 0.39%)</title><rect x="209.7" y="549" width="4.6" height="15.0" fill="rgb(248,20,53)" rx="2" ry="2" />
<text  x="212.67" y="559.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (60 samples, 0.02%)</title><rect x="1051.1" y="613" width="0.3" height="15.0" fill="rgb(232,155,2)" rx="2" ry="2" />
<text  x="1054.14" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,537 samples, 0.78%)</title><rect x="832.1" y="645" width="9.1" height="15.0" fill="rgb(230,146,24)" rx="2" ry="2" />
<text  x="835.07" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (123 samples, 0.04%)</title><rect x="157.4" y="549" width="0.4" height="15.0" fill="rgb(234,42,17)" rx="2" ry="2" />
<text  x="160.37" y="559.5" ></text>
</g>
<g >
<title>runtime.goexit (318,153 samples, 97.51%)</title><rect x="32.9" y="885" width="1150.6" height="15.0" fill="rgb(208,57,54)" rx="2" ry="2" />
<text  x="35.94" y="895.5" >runtime.goexit</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Close (383 samples, 0.12%)</title><rect x="64.8" y="693" width="1.4" height="15.0" fill="rgb(235,174,9)" rx="2" ry="2" />
<text  x="67.83" y="703.5" ></text>
</g>
<g >
<title>runtime.newobject (157 samples, 0.05%)</title><rect x="1086.4" y="725" width="0.6" height="15.0" fill="rgb(253,77,28)" rx="2" ry="2" />
<text  x="1089.40" y="735.5" ></text>
</g>
<g >
<title>runtime.schedule (32 samples, 0.01%)</title><rect x="16.5" y="805" width="0.1" height="15.0" fill="rgb(230,112,3)" rx="2" ry="2" />
<text  x="19.50" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (72 samples, 0.02%)</title><rect x="452.2" y="741" width="0.3" height="15.0" fill="rgb(206,110,44)" rx="2" ry="2" />
<text  x="455.24" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (353 samples, 0.11%)</title><rect x="1188.4" y="709" width="1.3" height="15.0" fill="rgb(215,136,3)" rx="2" ry="2" />
<text  x="1191.43" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="1083.3" y="357" width="0.2" height="15.0" fill="rgb(233,169,29)" rx="2" ry="2" />
<text  x="1086.31" y="367.5" ></text>
</g>
<g >
<title>ext4_reserve_inode_write (51 samples, 0.02%)</title><rect x="200.7" y="229" width="0.2" height="15.0" fill="rgb(216,61,51)" rx="2" ry="2" />
<text  x="203.69" y="239.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (560 samples, 0.17%)</title><rect x="1062.8" y="693" width="2.0" height="15.0" fill="rgb(245,29,32)" rx="2" ry="2" />
<text  x="1065.77" y="703.5" ></text>
</g>
<g >
<title>runtime.ready (66 samples, 0.02%)</title><rect x="28.9" y="869" width="0.3" height="15.0" fill="rgb(252,199,44)" rx="2" ry="2" />
<text  x="31.94" y="879.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="18.1" y="661" width="0.1" height="15.0" fill="rgb(219,141,2)" rx="2" ry="2" />
<text  x="21.05" y="671.5" ></text>
</g>
<g >
<title>runtime.futex (144 samples, 0.04%)</title><rect x="1108.5" y="741" width="0.5" height="15.0" fill="rgb(252,79,1)" rx="2" ry="2" />
<text  x="1111.52" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/syserr.TranslateNetstackError (40 samples, 0.01%)</title><rect x="53.3" y="725" width="0.2" height="15.0" fill="rgb(219,168,21)" rx="2" ry="2" />
<text  x="56.32" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportDemuxer).registerEndpoint (500 samples, 0.15%)</title><rect x="1022.5" y="805" width="1.8" height="15.0" fill="rgb(244,57,6)" rx="2" ry="2" />
<text  x="1025.50" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,753 samples, 0.54%)</title><rect x="834.9" y="597" width="6.3" height="15.0" fill="rgb(222,200,36)" rx="2" ry="2" />
<text  x="837.91" y="607.5" ></text>
</g>
<g >
<title>runtime.slicebytetostring (146 samples, 0.04%)</title><rect x="785.4" y="629" width="0.5" height="15.0" fill="rgb(209,64,28)" rx="2" ry="2" />
<text  x="788.39" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/sniffer.(*endpoint).DeliverOutboundPacket (52 samples, 0.02%)</title><rect x="1027.8" y="581" width="0.2" height="15.0" fill="rgb(241,80,11)" rx="2" ry="2" />
<text  x="1030.85" y="591.5" ></text>
</g>
<g >
<title>runtime.timeSleepUntil (70 samples, 0.02%)</title><rect x="14.0" y="757" width="0.2" height="15.0" fill="rgb(231,79,32)" rx="2" ry="2" />
<text  x="16.98" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MTU (28 samples, 0.01%)</title><rect x="1039.3" y="773" width="0.1" height="15.0" fill="rgb(223,172,21)" rx="2" ry="2" />
<text  x="1042.29" y="783.5" ></text>
</g>
<g >
<title>runtime.schedule (165 samples, 0.05%)</title><rect x="451.6" y="709" width="0.6" height="15.0" fill="rgb(223,134,35)" rx="2" ry="2" />
<text  x="454.64" y="719.5" ></text>
</g>
<g >
<title>runtime.entersyscall (60 samples, 0.02%)</title><rect x="230.3" y="773" width="0.3" height="15.0" fill="rgb(211,150,3)" rx="2" ry="2" />
<text  x="233.35" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).RUnlock (31 samples, 0.01%)</title><rect x="779.6" y="709" width="0.1" height="15.0" fill="rgb(236,168,20)" rx="2" ry="2" />
<text  x="782.56" y="719.5" ></text>
</g>
<g >
<title>time.startTimer (74 samples, 0.02%)</title><rect x="1069.0" y="821" width="0.3" height="15.0" fill="rgb(238,152,35)" rx="2" ry="2" />
<text  x="1072.05" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (60 samples, 0.02%)</title><rect x="184.0" y="661" width="0.3" height="15.0" fill="rgb(227,68,12)" rx="2" ry="2" />
<text  x="187.05" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (33 samples, 0.01%)</title><rect x="18.1" y="613" width="0.1" height="15.0" fill="rgb(205,155,6)" rx="2" ry="2" />
<text  x="21.06" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (33 samples, 0.01%)</title><rect x="1079.2" y="629" width="0.2" height="15.0" fill="rgb(211,115,49)" rx="2" ry="2" />
<text  x="1082.24" y="639.5" ></text>
</g>
<g >
<title>runtime.unlock2 (28 samples, 0.01%)</title><rect x="1003.4" y="709" width="0.1" height="15.0" fill="rgb(244,206,37)" rx="2" ry="2" />
<text  x="1006.37" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,808 samples, 0.86%)</title><rect x="685.5" y="645" width="10.1" height="15.0" fill="rgb(244,135,36)" rx="2" ry="2" />
<text  x="688.45" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).writePacket (716 samples, 0.22%)</title><rect x="1027.5" y="709" width="2.6" height="15.0" fill="rgb(247,116,29)" rx="2" ry="2" />
<text  x="1030.50" y="719.5" ></text>
</g>
<g >
<title>runtime.usleep (31 samples, 0.01%)</title><rect x="127.4" y="613" width="0.1" height="15.0" fill="rgb(217,49,27)" rx="2" ry="2" />
<text  x="130.37" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*fileOperations).maybeSync (28 samples, 0.01%)</title><rect x="202.8" y="693" width="0.1" height="15.0" fill="rgb(227,68,24)" rx="2" ry="2" />
<text  x="205.76" y="703.5" ></text>
</g>
<g >
<title>runtime.stringtoslicebyte (28 samples, 0.01%)</title><rect x="1093.6" y="725" width="0.1" height="15.0" fill="rgb(245,201,51)" rx="2" ry="2" />
<text  x="1096.64" y="735.5" ></text>
</g>
<g >
<title>runtime.step (120 samples, 0.04%)</title><rect x="1016.1" y="661" width="0.4" height="15.0" fill="rgb(241,200,54)" rx="2" ry="2" />
<text  x="1019.05" y="671.5" ></text>
</g>
<g >
<title>runtime.pidleput (75 samples, 0.02%)</title><rect x="116.7" y="629" width="0.2" height="15.0" fill="rgb(250,53,38)" rx="2" ry="2" />
<text  x="119.67" y="639.5" ></text>
</g>
<g >
<title>runtime.pcdatavalue (221 samples, 0.07%)</title><rect x="1015.7" y="693" width="0.8" height="15.0" fill="rgb(244,228,7)" rx="2" ry="2" />
<text  x="1018.73" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (39 samples, 0.01%)</title><rect x="192.1" y="629" width="0.2" height="15.0" fill="rgb(239,220,12)" rx="2" ry="2" />
<text  x="195.12" y="639.5" ></text>
</g>
<g >
<title>runtime.adjusttimers (1,085 samples, 0.33%)</title><rect x="108.6" y="613" width="3.9" height="15.0" fill="rgb(252,87,14)" rx="2" ry="2" />
<text  x="111.58" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip.(*Subnet).Broadcast (65 samples, 0.02%)</title><rect x="1036.5" y="645" width="0.2" height="15.0" fill="rgb(214,88,27)" rx="2" ry="2" />
<text  x="1039.46" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Get (106 samples, 0.03%)</title><rect x="174.2" y="741" width="0.4" height="15.0" fill="rgb(240,69,16)" rx="2" ry="2" />
<text  x="177.18" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).makeOptions (44 samples, 0.01%)</title><rect x="208.3" y="613" width="0.1" height="15.0" fill="rgb(229,45,31)" rx="2" ry="2" />
<text  x="211.29" y="623.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (55 samples, 0.02%)</title><rect x="13.8" y="613" width="0.2" height="15.0" fill="rgb(223,63,35)" rx="2" ry="2" />
<text  x="16.78" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/waiter.(*Queue).Notify (108 samples, 0.03%)</title><rect x="1071.5" y="725" width="0.4" height="15.0" fill="rgb(230,177,17)" rx="2" ry="2" />
<text  x="1074.47" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (76 samples, 0.02%)</title><rect x="1062.2" y="613" width="0.3" height="15.0" fill="rgb(235,78,49)" rx="2" ry="2" />
<text  x="1065.23" y="623.5" ></text>
</g>
<g >
<title>time.startTimer (463 samples, 0.14%)</title><rect x="1081.8" y="629" width="1.7" height="15.0" fill="rgb(226,95,11)" rx="2" ry="2" />
<text  x="1084.80" y="639.5" ></text>
</g>
<g >
<title>ext4_da_write_end (366 samples, 0.11%)</title><rect x="199.7" y="309" width="1.4" height="15.0" fill="rgb(227,79,40)" rx="2" ry="2" />
<text  x="202.74" y="319.5" ></text>
</g>
<g >
<title>runtime.stackcacherefill (783 samples, 0.24%)</title><rect x="1150.7" y="725" width="2.8" height="15.0" fill="rgb(217,137,20)" rx="2" ry="2" />
<text  x="1153.68" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).AddressSpace (30 samples, 0.01%)</title><rect x="223.6" y="821" width="0.1" height="15.0" fill="rgb(234,137,31)" rx="2" ry="2" />
<text  x="226.61" y="831.5" ></text>
</g>
<g >
<title>runtime.gentraceback (256 samples, 0.08%)</title><rect x="1183.7" y="837" width="1.0" height="15.0" fill="rgb(230,158,27)" rx="2" ry="2" />
<text  x="1186.73" y="847.5" ></text>
</g>
<g >
<title>runtime.mcall (62 samples, 0.02%)</title><rect x="54.4" y="709" width="0.3" height="15.0" fill="rgb(253,72,42)" rx="2" ry="2" />
<text  x="57.44" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).UnlockUser (50 samples, 0.02%)</title><rect x="54.3" y="709" width="0.1" height="15.0" fill="rgb(248,182,24)" rx="2" ry="2" />
<text  x="57.25" y="719.5" ></text>
</g>
<g >
<title>runtime.runOneTimer (119 samples, 0.04%)</title><rect x="903.8" y="709" width="0.4" height="15.0" fill="rgb(215,79,16)" rx="2" ry="2" />
<text  x="906.80" y="719.5" ></text>
</g>
<g >
<title>runtime.memmove (40 samples, 0.01%)</title><rect x="176.5" y="581" width="0.2" height="15.0" fill="rgb(249,60,15)" rx="2" ry="2" />
<text  x="179.51" y="591.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (130 samples, 0.04%)</title><rect x="13.5" y="709" width="0.5" height="15.0" fill="rgb(254,106,48)" rx="2" ry="2" />
<text  x="16.51" y="719.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (80 samples, 0.02%)</title><rect x="1108.7" y="613" width="0.3" height="15.0" fill="rgb(216,47,38)" rx="2" ry="2" />
<text  x="1111.75" y="623.5" ></text>
</g>
<g >
<title>aeshashbody (186 samples, 0.06%)</title><rect x="771.5" y="629" width="0.7" height="15.0" fill="rgb(215,148,12)" rx="2" ry="2" />
<text  x="774.52" y="639.5" ></text>
</g>
<g >
<title>runtime.futex (347 samples, 0.11%)</title><rect x="1127.3" y="757" width="1.3" height="15.0" fill="rgb(206,124,44)" rx="2" ry="2" />
<text  x="1130.31" y="767.5" ></text>
</g>
<g >
<title>runtime.checkTimers (73 samples, 0.02%)</title><rect x="1107.8" y="773" width="0.3" height="15.0" fill="rgb(234,110,3)" rx="2" ry="2" />
<text  x="1110.83" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyIn.func1 (41 samples, 0.01%)</title><rect x="56.8" y="645" width="0.2" height="15.0" fill="rgb(215,126,44)" rx="2" ry="2" />
<text  x="59.83" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*File).Flush (42 samples, 0.01%)</title><rect x="68.8" y="757" width="0.1" height="15.0" fill="rgb(228,13,7)" rx="2" ry="2" />
<text  x="71.78" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (34 samples, 0.01%)</title><rect x="1109.2" y="645" width="0.1" height="15.0" fill="rgb(207,0,54)" rx="2" ry="2" />
<text  x="1112.18" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*NIC).isValidForOutgoing (67 samples, 0.02%)</title><rect x="1093.1" y="709" width="0.3" height="15.0" fill="rgb(206,115,8)" rx="2" ry="2" />
<text  x="1096.12" y="719.5" ></text>
</g>
<g >
<title>runtime.gopark (31 samples, 0.01%)</title><rect x="697.6" y="853" width="0.1" height="15.0" fill="rgb(209,27,50)" rx="2" ry="2" />
<text  x="700.55" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).Getlink (111 samples, 0.03%)</title><rect x="189.1" y="693" width="0.4" height="15.0" fill="rgb(246,226,8)" rx="2" ry="2" />
<text  x="192.07" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Write (3,738 samples, 1.15%)</title><rect x="189.9" y="773" width="13.5" height="15.0" fill="rgb(206,229,19)" rx="2" ry="2" />
<text  x="192.88" y="783.5" ></text>
</g>
<g >
<title>runtime.newobject (96 samples, 0.03%)</title><rect x="1083.8" y="677" width="0.4" height="15.0" fill="rgb(253,63,7)" rx="2" ry="2" />
<text  x="1086.83" y="687.5" ></text>
</g>
<g >
<title>runtime.futex (57 samples, 0.02%)</title><rect x="65.8" y="517" width="0.2" height="15.0" fill="rgb(224,220,24)" rx="2" ry="2" />
<text  x="68.76" y="527.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs/gofer.(*inodeOperations).Check (209 samples, 0.06%)</title><rect x="188.1" y="645" width="0.8" height="15.0" fill="rgb(246,90,51)" rx="2" ry="2" />
<text  x="191.14" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (129 samples, 0.04%)</title><rect x="1083.0" y="469" width="0.5" height="15.0" fill="rgb(227,166,34)" rx="2" ry="2" />
<text  x="1086.00" y="479.5" ></text>
</g>
<g >
<title>runtime.goexit0 (32 samples, 0.01%)</title><rect x="16.5" y="821" width="0.1" height="15.0" fill="rgb(217,56,51)" rx="2" ry="2" />
<text  x="19.50" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).MaxHeaderLength (38 samples, 0.01%)</title><rect x="1026.9" y="741" width="0.1" height="15.0" fill="rgb(216,50,17)" rx="2" ry="2" />
<text  x="1029.91" y="751.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc1 (48 samples, 0.01%)</title><rect x="929.1" y="741" width="0.2" height="15.0" fill="rgb(240,198,5)" rx="2" ry="2" />
<text  x="932.08" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (87 samples, 0.03%)</title><rect x="1064.5" y="517" width="0.3" height="15.0" fill="rgb(218,194,13)" rx="2" ry="2" />
<text  x="1067.48" y="527.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).internalMappingsLocked (33 samples, 0.01%)</title><rect x="62.9" y="693" width="0.1" height="15.0" fill="rgb(232,178,10)" rx="2" ry="2" />
<text  x="65.89" y="703.5" ></text>
</g>
<g >
<title>runtime.mallocgc (271 samples, 0.08%)</title><rect x="55.4" y="709" width="1.0" height="15.0" fill="rgb(254,205,25)" rx="2" ry="2" />
<text  x="58.37" y="719.5" ></text>
</g>
<g >
<title>runtime.newMarkBits (37 samples, 0.01%)</title><rect x="1110.4" y="821" width="0.1" height="15.0" fill="rgb(222,226,4)" rx="2" ry="2" />
<text  x="1113.36" y="831.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*FDTable).Get (29 samples, 0.01%)</title><rect x="43.3" y="757" width="0.1" height="15.0" fill="rgb(205,110,9)" rx="2" ry="2" />
<text  x="46.31" y="767.5" ></text>
</g>
<g >
<title>runtime.systemstack (57 samples, 0.02%)</title><rect x="1111.0" y="837" width="0.2" height="15.0" fill="rgb(223,29,11)" rx="2" ry="2" />
<text  x="1114.04" y="847.5" ></text>
</g>
<g >
<title>runtime.usleep (478 samples, 0.15%)</title><rect x="1121.4" y="773" width="1.7" height="15.0" fill="rgb(226,228,32)" rx="2" ry="2" />
<text  x="1124.42" y="783.5" ></text>
</g>
<g >
<title>runtime.stackpoolfree (34 samples, 0.01%)</title><rect x="1104.7" y="693" width="0.1" height="15.0" fill="rgb(227,161,51)" rx="2" ry="2" />
<text  x="1107.69" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).withInternalMappings (741 samples, 0.23%)</title><rect x="74.0" y="709" width="2.7" height="15.0" fill="rgb(239,115,14)" rx="2" ry="2" />
<text  x="76.99" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).addIPHeader (60 samples, 0.02%)</title><rect x="1090.8" y="725" width="0.2" height="15.0" fill="rgb(235,131,38)" rx="2" ry="2" />
<text  x="1093.80" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="1153.4" y="629" width="0.1" height="15.0" fill="rgb(240,195,19)" rx="2" ry="2" />
<text  x="1156.36" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/safemem.CopySeq (176 samples, 0.05%)</title><rect x="61.9" y="677" width="0.7" height="15.0" fill="rgb(208,171,47)" rx="2" ry="2" />
<text  x="64.94" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.NewMountSource (145 samples, 0.04%)</title><rect x="51.3" y="693" width="0.5" height="15.0" fill="rgb(249,33,2)" rx="2" ry="2" />
<text  x="54.28" y="703.5" ></text>
</g>
<g >
<title>runtime.funcMaxSPDelta (45 samples, 0.01%)</title><rect x="1184.8" y="853" width="0.2" height="15.0" fill="rgb(223,39,24)" rx="2" ry="2" />
<text  x="1187.81" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (77 samples, 0.02%)</title><rect x="162.2" y="693" width="0.2" height="15.0" fill="rgb(249,14,32)" rx="2" ry="2" />
<text  x="165.17" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (9,566 samples, 2.93%)</title><rect x="661.0" y="757" width="34.6" height="15.0" fill="rgb(205,96,33)" rx="2" ry="2" />
<text  x="664.02" y="767.5" >[[..</text>
</g>
<g >
<title>runtime.startm (87 samples, 0.03%)</title><rect x="1005.8" y="789" width="0.3" height="15.0" fill="rgb(251,30,2)" rx="2" ry="2" />
<text  x="1008.77" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/mm.(*MemoryManager).CopyOut (39 samples, 0.01%)</title><rect x="76.8" y="741" width="0.1" height="15.0" fill="rgb(237,36,10)" rx="2" ry="2" />
<text  x="79.81" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindNICNameFromID (29 samples, 0.01%)</title><rect x="1075.4" y="677" width="0.1" height="15.0" fill="rgb(209,173,12)" rx="2" ry="2" />
<text  x="1078.39" y="687.5" ></text>
</g>
<g >
<title>br_nf_hook_thresh (8,190 samples, 2.51%)</title><rect x="965.5" y="469" width="29.6" height="15.0" fill="rgb(239,211,11)" rx="2" ry="2" />
<text  x="968.52" y="479.5" >br..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Sleeper).AddWaker (49 samples, 0.02%)</title><rect x="1045.3" y="853" width="0.2" height="15.0" fill="rgb(240,100,50)" rx="2" ry="2" />
<text  x="1048.30" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (7,095 samples, 2.17%)</title><rect x="869.2" y="773" width="25.7" height="15.0" fill="rgb(237,69,9)" rx="2" ry="2" />
<text  x="872.23" y="783.5" >[..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (96 samples, 0.03%)</title><rect x="198.8" y="277" width="0.4" height="15.0" fill="rgb(209,17,24)" rx="2" ry="2" />
<text  x="201.83" y="287.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/network/ipv4.(*endpoint).AcquireAssignedAddress.func1 (279 samples, 0.09%)</title><rect x="734.9" y="677" width="1.0" height="15.0" fill="rgb(241,44,52)" rx="2" ry="2" />
<text  x="737.87" y="687.5" ></text>
</g>
<g >
<title>ip_sabotage_in (7,227 samples, 2.21%)</title><rect x="968.4" y="357" width="26.2" height="15.0" fill="rgb(207,111,25)" rx="2" ry="2" />
<text  x="971.44" y="367.5" >i..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.sendTCP (1,338 samples, 0.41%)</title><rect x="1026.5" y="757" width="4.9" height="15.0" fill="rgb(215,57,8)" rx="2" ry="2" />
<text  x="1029.54" y="767.5" ></text>
</g>
<g >
<title>runtime.addtimer (266 samples, 0.08%)</title><rect x="1048.8" y="677" width="0.9" height="15.0" fill="rgb(223,70,6)" rx="2" ry="2" />
<text  x="1051.78" y="687.5" ></text>
</g>
<g >
<title>runtime.runqsteal (244 samples, 0.07%)</title><rect x="1061.6" y="741" width="0.9" height="15.0" fill="rgb(218,36,36)" rx="2" ry="2" />
<text  x="1064.62" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/refs.(*WeakRef).get (43 samples, 0.01%)</title><rect x="67.3" y="677" width="0.2" height="15.0" fill="rgb(216,104,21)" rx="2" ry="2" />
<text  x="70.32" y="687.5" ></text>
</g>
<g >
<title>runtime.mcall (56 samples, 0.02%)</title><rect x="1021.6" y="821" width="0.2" height="15.0" fill="rgb(252,120,15)" rx="2" ry="2" />
<text  x="1024.56" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (67 samples, 0.02%)</title><rect x="1049.5" y="485" width="0.2" height="15.0" fill="rgb(237,181,33)" rx="2" ry="2" />
<text  x="1052.50" y="495.5" ></text>
</g>
<g >
<title>br_forward_finish (237 samples, 0.07%)</title><rect x="989.9" y="133" width="0.9" height="15.0" fill="rgb(213,64,21)" rx="2" ry="2" />
<text  x="992.93" y="143.5" ></text>
</g>
<g >
<title>runtime.(*mcache).prepareForSweep (28 samples, 0.01%)</title><rect x="843.8" y="741" width="0.1" height="15.0" fill="rgb(206,176,50)" rx="2" ry="2" />
<text  x="846.77" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel/time.(*Timer).SwapAnd (96 samples, 0.03%)</title><rect x="1079.2" y="661" width="0.4" height="15.0" fill="rgb(223,198,27)" rx="2" ry="2" />
<text  x="1082.22" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).Capabilities (39 samples, 0.01%)</title><rect x="738.9" y="661" width="0.1" height="15.0" fill="rgb(212,212,28)" rx="2" ry="2" />
<text  x="741.88" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/qdisc/fifo.(*endpoint).WritePacket (106 samples, 0.03%)</title><rect x="1091.6" y="613" width="0.3" height="15.0" fill="rgb(207,149,11)" rx="2" ry="2" />
<text  x="1094.56" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Timekeeper).GetTime (92 samples, 0.03%)</title><rect x="1085.8" y="661" width="0.3" height="15.0" fill="rgb(225,163,48)" rx="2" ry="2" />
<text  x="1088.81" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyInBytes (128 samples, 0.04%)</title><rect x="203.8" y="725" width="0.5" height="15.0" fill="rgb(238,140,47)" rx="2" ry="2" />
<text  x="206.84" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Stack).FindNICNameFromID (29 samples, 0.01%)</title><rect x="214.4" y="549" width="0.1" height="15.0" fill="rgb(250,134,5)" rx="2" ry="2" />
<text  x="217.42" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*handshake).resetState (70 samples, 0.02%)</title><rect x="1024.8" y="773" width="0.3" height="15.0" fill="rgb(239,185,22)" rx="2" ry="2" />
<text  x="1027.81" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.muldiv64 (31 samples, 0.01%)</title><rect x="60.5" y="693" width="0.1" height="15.0" fill="rgb(214,116,46)" rx="2" ry="2" />
<text  x="63.48" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).transitionToStateEstablishedLocked (1,419 samples, 0.43%)</title><rect x="1008.3" y="789" width="5.1" height="15.0" fill="rgb(206,155,45)" rx="2" ry="2" />
<text  x="1011.29" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (103 samples, 0.03%)</title><rect x="1128.2" y="613" width="0.4" height="15.0" fill="rgb(242,196,6)" rx="2" ry="2" />
<text  x="1131.19" y="623.5" ></text>
</g>
<g >
<title>reflect.Value.Field (31 samples, 0.01%)</title><rect x="925.6" y="789" width="0.2" height="15.0" fill="rgb(251,138,43)" rx="2" ry="2" />
<text  x="928.65" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (43 samples, 0.01%)</title><rect x="1150.9" y="629" width="0.1" height="15.0" fill="rgb(237,163,26)" rx="2" ry="2" />
<text  x="1153.87" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*transportDemuxer).unregisterEndpoint (1,136 samples, 0.35%)</title><rect x="1101.1" y="805" width="4.1" height="15.0" fill="rgb(238,169,27)" rx="2" ry="2" />
<text  x="1104.06" y="815.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.ParseTCPOptions (46 samples, 0.01%)</title><rect x="759.2" y="629" width="0.1" height="15.0" fill="rgb(254,61,10)" rx="2" ry="2" />
<text  x="762.17" y="639.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*thread).syscall (328 samples, 0.10%)</title><rect x="703.9" y="821" width="1.1" height="15.0" fill="rgb(235,20,11)" rx="2" ry="2" />
<text  x="706.85" y="831.5" ></text>
</g>
<g >
<title>runtime.schedule (532 samples, 0.16%)</title><rect x="21.8" y="837" width="1.9" height="15.0" fill="rgb(206,84,25)" rx="2" ry="2" />
<text  x="24.81" y="847.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (175 samples, 0.05%)</title><rect x="1082.8" y="501" width="0.7" height="15.0" fill="rgb(236,163,3)" rx="2" ry="2" />
<text  x="1085.83" y="511.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,001 samples, 0.61%)</title><rect x="750.8" y="469" width="7.3" height="15.0" fill="rgb(228,132,47)" rx="2" ry="2" />
<text  x="753.83" y="479.5" ></text>
</g>
<g >
<title>runtime.notewakeup (716 samples, 0.22%)</title><rect x="919.3" y="709" width="2.6" height="15.0" fill="rgb(206,186,8)" rx="2" ry="2" />
<text  x="922.30" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*syscallTSCReferenceClocks).Cycles (33 samples, 0.01%)</title><rect x="60.8" y="709" width="0.1" height="15.0" fill="rgb(216,50,25)" rx="2" ry="2" />
<text  x="63.75" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/platform/ptrace.(*subprocess).newThread-fm (253 samples, 0.08%)</title><rect x="451.6" y="789" width="0.9" height="15.0" fill="rgb(218,174,15)" rx="2" ry="2" />
<text  x="454.59" y="799.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (54 samples, 0.02%)</title><rect x="149.2" y="485" width="0.2" height="15.0" fill="rgb(209,46,37)" rx="2" ry="2" />
<text  x="152.15" y="495.5" ></text>
</g>
<g >
<title>runtime.stackpoolalloc (248 samples, 0.08%)</title><rect x="1152.4" y="709" width="0.9" height="15.0" fill="rgb(249,8,28)" rx="2" ry="2" />
<text  x="1155.36" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).transitionToStateCloseLocked (1,604 samples, 0.49%)</title><rect x="1099.6" y="853" width="5.8" height="15.0" fill="rgb(222,14,45)" rx="2" ry="2" />
<text  x="1102.63" y="863.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (160 samples, 0.05%)</title><rect x="451.7" y="629" width="0.5" height="15.0" fill="rgb(241,57,35)" rx="2" ry="2" />
<text  x="454.66" y="639.5" ></text>
</g>
<g >
<title>runtime.(*mcache).refill (55 samples, 0.02%)</title><rect x="1012.8" y="709" width="0.2" height="15.0" fill="rgb(228,49,15)" rx="2" ry="2" />
<text  x="1015.82" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).entry (53 samples, 0.02%)</title><rect x="1092.3" y="645" width="0.2" height="15.0" fill="rgb(228,181,46)" rx="2" ry="2" />
<text  x="1095.30" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (28 samples, 0.01%)</title><rect x="1051.3" y="533" width="0.1" height="15.0" fill="rgb(222,212,51)" rx="2" ry="2" />
<text  x="1054.25" y="543.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sleep.(*Waker).Assert (30 samples, 0.01%)</title><rect x="1008.0" y="821" width="0.1" height="15.0" fill="rgb(227,167,8)" rx="2" ry="2" />
<text  x="1011.04" y="831.5" ></text>
</g>
<g >
<title>runtime.findrunnable (5,689 samples, 1.74%)</title><rect x="898.4" y="757" width="20.5" height="15.0" fill="rgb(211,93,4)" rx="2" ry="2" />
<text  x="901.36" y="767.5" ></text>
</g>
<g >
<title>runtime.mallocgc (48 samples, 0.01%)</title><rect x="50.0" y="645" width="0.2" height="15.0" fill="rgb(233,133,3)" rx="2" ry="2" />
<text  x="53.02" y="655.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (92 samples, 0.03%)</title><rect x="148.1" y="389" width="0.3" height="15.0" fill="rgb(240,196,13)" rx="2" ry="2" />
<text  x="151.08" y="399.5" ></text>
</g>
<g >
<title>runtime.(*pageAlloc).scavengeRangeLocked (223 samples, 0.07%)</title><rect x="1130.1" y="805" width="0.8" height="15.0" fill="rgb(210,137,50)" rx="2" ry="2" />
<text  x="1133.10" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (39 samples, 0.01%)</title><rect x="699.9" y="725" width="0.2" height="15.0" fill="rgb(252,0,0)" rx="2" ry="2" />
<text  x="702.94" y="735.5" ></text>
</g>
<g >
<title>runtime.pcvalue (238 samples, 0.07%)</title><rect x="1148.9" y="677" width="0.9" height="15.0" fill="rgb(218,25,29)" rx="2" ry="2" />
<text  x="1151.93" y="687.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (71 samples, 0.02%)</title><rect x="904.6" y="597" width="0.3" height="15.0" fill="rgb(206,140,43)" rx="2" ry="2" />
<text  x="907.65" y="607.5" ></text>
</g>
<g >
<title>runtime.duffzero (40 samples, 0.01%)</title><rect x="34.5" y="837" width="0.1" height="15.0" fill="rgb(224,197,49)" rx="2" ry="2" />
<text  x="37.50" y="847.5" ></text>
</g>
<g >
<title>runtime.stackfree (53 samples, 0.02%)</title><rect x="1104.6" y="725" width="0.2" height="15.0" fill="rgb(229,80,45)" rx="2" ry="2" />
<text  x="1107.62" y="735.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (631 samples, 0.19%)</title><rect x="210.8" y="277" width="2.3" height="15.0" fill="rgb(218,159,7)" rx="2" ry="2" />
<text  x="213.83" y="287.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (100 samples, 0.03%)</title><rect x="13.0" y="661" width="0.4" height="15.0" fill="rgb(227,192,24)" rx="2" ry="2" />
<text  x="16.03" y="671.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="65.8" y="469" width="0.2" height="15.0" fill="rgb(216,90,33)" rx="2" ry="2" />
<text  x="68.84" y="479.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (90 samples, 0.03%)</title><rect x="1152.8" y="645" width="0.3" height="15.0" fill="rgb(215,110,29)" rx="2" ry="2" />
<text  x="1155.78" y="655.5" ></text>
</g>
<g >
<title>runtime.runqput (57 samples, 0.02%)</title><rect x="1003.7" y="677" width="0.2" height="15.0" fill="rgb(223,29,1)" rx="2" ry="2" />
<text  x="1006.73" y="687.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (52 samples, 0.02%)</title><rect x="726.4" y="661" width="0.2" height="15.0" fill="rgb(229,18,36)" rx="2" ry="2" />
<text  x="729.38" y="671.5" ></text>
</g>
<g >
<title>runtime.selectnbrecv (68 samples, 0.02%)</title><rect x="161.4" y="709" width="0.2" height="15.0" fill="rgb(239,21,16)" rx="2" ry="2" />
<text  x="164.35" y="719.5" ></text>
</g>
<g >
<title>runtime.makechan (53 samples, 0.02%)</title><rect x="1005.0" y="837" width="0.2" height="15.0" fill="rgb(249,176,42)" rx="2" ry="2" />
<text  x="1008.04" y="847.5" ></text>
</g>
<g >
<title>runtime.runqsteal (1,411 samples, 0.43%)</title><rect x="905.8" y="741" width="5.1" height="15.0" fill="rgb(248,148,17)" rx="2" ry="2" />
<text  x="908.76" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).UnstableAttr (113 samples, 0.03%)</title><rect x="188.2" y="613" width="0.4" height="15.0" fill="rgb(211,211,23)" rx="2" ry="2" />
<text  x="191.22" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).Write.func1 (648 samples, 0.20%)</title><rect x="205.4" y="693" width="2.4" height="15.0" fill="rgb(249,2,19)" rx="2" ry="2" />
<text  x="208.44" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (59 samples, 0.02%)</title><rect x="997.9" y="597" width="0.2" height="15.0" fill="rgb(231,80,0)" rx="2" ry="2" />
<text  x="1000.92" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (6,790 samples, 2.08%)</title><rect x="797.6" y="805" width="24.6" height="15.0" fill="rgb(245,229,10)" rx="2" ry="2" />
<text  x="800.61" y="815.5" >[..</text>
</g>
<g >
<title>[[kernel.kallsyms]] (661 samples, 0.20%)</title><rect x="908.4" y="661" width="2.4" height="15.0" fill="rgb(233,195,38)" rx="2" ry="2" />
<text  x="911.39" y="671.5" ></text>
</g>
<g >
<title>runtime.doaddtimer (43 samples, 0.01%)</title><rect x="1048.9" y="661" width="0.1" height="15.0" fill="rgb(254,68,33)" rx="2" ry="2" />
<text  x="1051.86" y="671.5" ></text>
</g>
<g >
<title>runtime.notewakeup (105 samples, 0.03%)</title><rect x="23.3" y="773" width="0.4" height="15.0" fill="rgb(220,189,36)" rx="2" ry="2" />
<text  x="26.32" y="783.5" ></text>
</g>
<g >
<title>runtime.mallocgc (105 samples, 0.03%)</title><rect x="207.3" y="645" width="0.4" height="15.0" fill="rgb(216,123,21)" rx="2" ry="2" />
<text  x="210.30" y="655.5" ></text>
</g>
<g >
<title>syscall.RawSyscall6 (18,712 samples, 5.73%)</title><rect x="930.5" y="805" width="67.6" height="15.0" fill="rgb(233,128,17)" rx="2" ry="2" />
<text  x="933.47" y="815.5" >syscall..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*CrossGoroutineRWMutex).Lock (121 samples, 0.04%)</title><rect x="782.8" y="693" width="0.4" height="15.0" fill="rgb(222,203,19)" rx="2" ry="2" />
<text  x="785.76" y="703.5" ></text>
</g>
<g >
<title>runtime.newproc1 (70 samples, 0.02%)</title><rect x="114.1" y="533" width="0.3" height="15.0" fill="rgb(247,102,18)" rx="2" ry="2" />
<text  x="117.14" y="543.5" ></text>
</g>
<g >
<title>runtime.putempty (156 samples, 0.05%)</title><rect x="1133.2" y="757" width="0.6" height="15.0" fill="rgb(211,42,23)" rx="2" ry="2" />
<text  x="1136.21" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (9,243 samples, 2.83%)</title><rect x="662.2" y="741" width="33.4" height="15.0" fill="rgb(239,37,38)" rx="2" ry="2" />
<text  x="665.19" y="751.5" >[[..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header/parse.TCP (33 samples, 0.01%)</title><rect x="722.6" y="741" width="0.1" height="15.0" fill="rgb(218,46,29)" rx="2" ry="2" />
<text  x="725.60" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,432 samples, 0.44%)</title><rect x="752.9" y="405" width="5.2" height="15.0" fill="rgb(206,85,50)" rx="2" ry="2" />
<text  x="755.89" y="415.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/time.(*CalibratedClocks).GetTime (50 samples, 0.02%)</title><rect x="162.3" y="677" width="0.1" height="15.0" fill="rgb(226,171,46)" rx="2" ry="2" />
<text  x="165.27" y="687.5" ></text>
</g>
<g >
<title>runtime.usleep (1,320 samples, 0.40%)</title><rect x="122.6" y="597" width="4.8" height="15.0" fill="rgb(216,124,48)" rx="2" ry="2" />
<text  x="125.60" y="607.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*timer).enable (87 samples, 0.03%)</title><rect x="216.0" y="661" width="0.3" height="15.0" fill="rgb(252,159,14)" rx="2" ry="2" />
<text  x="218.96" y="671.5" ></text>
</g>
<g >
<title>runtime.siftupTimer (31 samples, 0.01%)</title><rect x="1048.9" y="645" width="0.1" height="15.0" fill="rgb(248,113,53)" rx="2" ry="2" />
<text  x="1051.90" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/link/fdbased.(*iovecBuffer).pullViews (1,495 samples, 0.46%)</title><rect x="712.2" y="821" width="5.4" height="15.0" fill="rgb(226,11,2)" rx="2" ry="2" />
<text  x="715.23" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (2,584 samples, 0.79%)</title><rect x="831.9" y="661" width="9.3" height="15.0" fill="rgb(212,44,13)" rx="2" ry="2" />
<text  x="834.90" y="671.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*handshake).complete (3,967 samples, 1.22%)</title><rect x="1006.1" y="853" width="14.3" height="15.0" fill="rgb(228,82,40)" rx="2" ry="2" />
<text  x="1009.09" y="863.5" ></text>
</g>
<g >
<title>runtime.wakep (107 samples, 0.03%)</title><rect x="19.0" y="853" width="0.4" height="15.0" fill="rgb(208,56,13)" rx="2" ry="2" />
<text  x="22.03" y="863.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.(*endpoint).handleClose (893 samples, 0.27%)</title><rect x="179.8" y="709" width="3.2" height="15.0" fill="rgb(229,48,18)" rx="2" ry="2" />
<text  x="182.80" y="719.5" ></text>
</g>
<g >
<title>runtime.systemstack (39 samples, 0.01%)</title><rect x="1074.1" y="517" width="0.1" height="15.0" fill="rgb(239,134,30)" rx="2" ry="2" />
<text  x="1077.08" y="527.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (47 samples, 0.01%)</title><rect x="1150.9" y="661" width="0.1" height="15.0" fill="rgb(207,158,54)" rx="2" ry="2" />
<text  x="1153.86" y="671.5" ></text>
</g>
<g >
<title>runtime.mallocgc (95 samples, 0.03%)</title><rect x="740.2" y="661" width="0.3" height="15.0" fill="rgb(240,192,16)" rx="2" ry="2" />
<text  x="743.17" y="671.5" ></text>
</g>
<g >
<title>runtime.casgstatus (63 samples, 0.02%)</title><rect x="747.1" y="549" width="0.3" height="15.0" fill="rgb(247,22,21)" rx="2" ry="2" />
<text  x="750.14" y="559.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.recvFrom (1,509 samples, 0.46%)</title><rect x="173.9" y="757" width="5.4" height="15.0" fill="rgb(226,44,37)" rx="2" ry="2" />
<text  x="176.86" y="767.5" ></text>
</g>
<g >
<title>runtime.mallocgc (43 samples, 0.01%)</title><rect x="1050.0" y="725" width="0.1" height="15.0" fill="rgb(251,109,19)" rx="2" ry="2" />
<text  x="1052.98" y="735.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/buffer.(*VectorisedView).PullUp (28 samples, 0.01%)</title><rect x="709.5" y="821" width="0.1" height="15.0" fill="rgb(223,147,8)" rx="2" ry="2" />
<text  x="712.54" y="831.5" ></text>
</g>
<g >
<title>runtime.(*mheap).allocSpan (43 samples, 0.01%)</title><rect x="1018.1" y="693" width="0.1" height="15.0" fill="rgb(221,143,50)" rx="2" ry="2" />
<text  x="1021.08" y="703.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/transport/tcp.newSender (1,200 samples, 0.37%)</title><rect x="1009.0" y="773" width="4.4" height="15.0" fill="rgb(208,59,47)" rx="2" ry="2" />
<text  x="1012.01" y="783.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).WritePacket (911 samples, 0.28%)</title><rect x="1027.2" y="741" width="3.2" height="15.0" fill="rgb(219,183,3)" rx="2" ry="2" />
<text  x="1030.15" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (565 samples, 0.17%)</title><rect x="195.9" y="357" width="2.0" height="15.0" fill="rgb(242,81,32)" rx="2" ry="2" />
<text  x="198.88" y="367.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (152 samples, 0.05%)</title><rect x="1082.9" y="485" width="0.6" height="15.0" fill="rgb(230,124,54)" rx="2" ry="2" />
<text  x="1085.92" y="495.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*Mutex).Unlock (47 samples, 0.01%)</title><rect x="166.5" y="693" width="0.1" height="15.0" fill="rgb(242,20,3)" rx="2" ry="2" />
<text  x="169.47" y="703.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (36 samples, 0.01%)</title><rect x="65.8" y="453" width="0.2" height="15.0" fill="rgb(236,13,35)" rx="2" ry="2" />
<text  x="68.84" y="463.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (62 samples, 0.02%)</title><rect x="1189.8" y="613" width="0.2" height="15.0" fill="rgb(229,200,0)" rx="2" ry="2" />
<text  x="1192.77" y="623.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/header.PseudoHeaderChecksum (80 samples, 0.02%)</title><rect x="1093.4" y="741" width="0.3" height="15.0" fill="rgb(245,129,35)" rx="2" ry="2" />
<text  x="1096.45" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/kernel.(*Task).CopyOutBytes (40 samples, 0.01%)</title><rect x="77.1" y="757" width="0.2" height="15.0" fill="rgb(219,152,52)" rx="2" ry="2" />
<text  x="80.11" y="767.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (61 samples, 0.02%)</title><rect x="452.3" y="661" width="0.2" height="15.0" fill="rgb(246,182,36)" rx="2" ry="2" />
<text  x="455.28" y="671.5" ></text>
</g>
<g >
<title>runtime.newobject (101 samples, 0.03%)</title><rect x="1094.1" y="757" width="0.4" height="15.0" fill="rgb(234,84,42)" rx="2" ry="2" />
<text  x="1097.11" y="767.5" ></text>
</g>
<g >
<title>runtime.gcAssistAlloc.func1 (28 samples, 0.01%)</title><rect x="1068.3" y="773" width="0.1" height="15.0" fill="rgb(249,118,22)" rx="2" ry="2" />
<text  x="1071.34" y="783.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (54 samples, 0.02%)</title><rect x="994.9" y="421" width="0.2" height="15.0" fill="rgb(241,108,49)" rx="2" ry="2" />
<text  x="997.94" y="431.5" ></text>
</g>
<g >
<title>jbd2_journal_stop (49 samples, 0.02%)</title><rect x="200.9" y="277" width="0.2" height="15.0" fill="rgb(216,92,9)" rx="2" ry="2" />
<text  x="203.89" y="287.5" ></text>
</g>
<g >
<title>runtime.(*mcache).nextFree (34 samples, 0.01%)</title><rect x="930.0" y="773" width="0.1" height="15.0" fill="rgb(211,196,24)" rx="2" ry="2" />
<text  x="932.96" y="783.5" ></text>
</g>
<g >
<title>runtime.exitsyscallfast.func1 (446 samples, 0.14%)</title><rect x="843.4" y="773" width="1.6" height="15.0" fill="rgb(214,127,20)" rx="2" ry="2" />
<text  x="846.37" y="783.5" ></text>
</g>
<g >
<title>runtime.gcDrain (14,483 samples, 4.44%)</title><rect x="1131.0" y="837" width="52.3" height="15.0" fill="rgb(253,83,23)" rx="2" ry="2" />
<text  x="1133.96" y="847.5" >runti..</text>
</g>
<g >
<title>ext4_da_write_begin (287 samples, 0.09%)</title><rect x="198.7" y="309" width="1.0" height="15.0" fill="rgb(248,22,40)" rx="2" ry="2" />
<text  x="201.71" y="319.5" ></text>
</g>
<g >
<title>runtime.(*mcentral).cacheSpan (41 samples, 0.01%)</title><rect x="207.4" y="597" width="0.2" height="15.0" fill="rgb(214,55,51)" rx="2" ry="2" />
<text  x="210.41" y="607.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (58 samples, 0.02%)</title><rect x="149.1" y="501" width="0.3" height="15.0" fill="rgb(228,0,16)" rx="2" ry="2" />
<text  x="152.14" y="511.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*Route).ConfirmReachable (2,595 samples, 0.80%)</title><rect x="1046.8" y="853" width="9.4" height="15.0" fill="rgb(221,209,9)" rx="2" ry="2" />
<text  x="1049.84" y="863.5" ></text>
</g>
<g >
<title>runtime.dodeltimer0 (87 samples, 0.03%)</title><rect x="1046.1" y="741" width="0.4" height="15.0" fill="rgb(206,180,35)" rx="2" ry="2" />
<text  x="1049.15" y="751.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sync.(*RWMutex).Unlock (61 samples, 0.02%)</title><rect x="784.4" y="677" width="0.2" height="15.0" fill="rgb(210,179,6)" rx="2" ry="2" />
<text  x="787.40" y="687.5" ></text>
</g>
<g >
<title>runtime.(*stackScanState).putPtr (269 samples, 0.08%)</title><rect x="1142.9" y="709" width="0.9" height="15.0" fill="rgb(208,143,40)" rx="2" ry="2" />
<text  x="1145.87" y="719.5" ></text>
</g>
<g >
<title>runtime.makeslice (386 samples, 0.12%)</title><rect x="927.9" y="805" width="1.4" height="15.0" fill="rgb(226,137,48)" rx="2" ry="2" />
<text  x="930.92" y="815.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (41 samples, 0.01%)</title><rect x="904.8" y="581" width="0.1" height="15.0" fill="rgb(211,192,21)" rx="2" ry="2" />
<text  x="907.75" y="591.5" ></text>
</g>
<g >
<title>runtime.typedslicecopy (43 samples, 0.01%)</title><rect x="850.3" y="821" width="0.2" height="15.0" fill="rgb(247,196,8)" rx="2" ry="2" />
<text  x="853.31" y="831.5" ></text>
</g>
<g >
<title>runtime.UnlockOSThread (36 samples, 0.01%)</title><rect x="639.8" y="821" width="0.1" height="15.0" fill="rgb(213,228,54)" rx="2" ry="2" />
<text  x="642.77" y="831.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (45 samples, 0.01%)</title><rect x="1153.4" y="645" width="0.1" height="15.0" fill="rgb(226,136,54)" rx="2" ry="2" />
<text  x="1156.35" y="655.5" ></text>
</g>
<g >
<title>runtime.runtimer (30 samples, 0.01%)</title><rect x="1108.0" y="757" width="0.1" height="15.0" fill="rgb(209,14,39)" rx="2" ry="2" />
<text  x="1110.98" y="767.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/fs.(*Inode).Getlink (56 samples, 0.02%)</title><rect x="189.1" y="661" width="0.3" height="15.0" fill="rgb(236,127,44)" rx="2" ry="2" />
<text  x="192.15" y="671.5" ></text>
</g>
<g >
<title>runtime.makeslice (38 samples, 0.01%)</title><rect x="777.4" y="709" width="0.1" height="15.0" fill="rgb(206,114,1)" rx="2" ry="2" />
<text  x="780.37" y="719.5" ></text>
</g>
<g >
<title>runtime.(*itabTableType).find (37 samples, 0.01%)</title><rect x="1038.5" y="725" width="0.1" height="15.0" fill="rgb(237,50,39)" rx="2" ry="2" />
<text  x="1041.47" y="735.5" ></text>
</g>
<g >
<title>veth_xmit (85 samples, 0.03%)</title><rect x="990.5" y="69" width="0.3" height="15.0" fill="rgb(234,47,37)" rx="2" ry="2" />
<text  x="993.48" y="79.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.Writev (3,749 samples, 1.15%)</title><rect x="203.4" y="773" width="13.6" height="15.0" fill="rgb(231,24,30)" rx="2" ry="2" />
<text  x="206.40" y="783.5" ></text>
</g>
<g >
<title>runtime.makeslice (49 samples, 0.02%)</title><rect x="1076.2" y="709" width="0.2" height="15.0" fill="rgb(207,164,38)" rx="2" ry="2" />
<text  x="1079.18" y="719.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*PacketBuffer).push (31 samples, 0.01%)</title><rect x="926.2" y="789" width="0.1" height="15.0" fill="rgb(249,148,37)" rx="2" ry="2" />
<text  x="929.20" y="799.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/sentry/syscalls/linux.ClockGettime (1,238 samples, 0.38%)</title><rect x="58.9" y="773" width="4.5" height="15.0" fill="rgb(229,169,28)" rx="2" ry="2" />
<text  x="61.93" y="783.5" ></text>
</g>
<g >
<title>runtime.newobject (43 samples, 0.01%)</title><rect x="1079.9" y="645" width="0.2" height="15.0" fill="rgb(223,109,41)" rx="2" ry="2" />
<text  x="1082.90" y="655.5" ></text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*addressState).IsAssigned (291 samples, 0.09%)</title><rect x="736.4" y="677" width="1.0" height="15.0" fill="rgb(234,212,49)" rx="2" ry="2" />
<text  x="739.35" y="687.5" ></text>
</g>
<g >
<title>runtime.funcMaxSPDelta (41 samples, 0.01%)</title><rect x="1055.4" y="741" width="0.2" height="15.0" fill="rgb(230,154,30)" rx="2" ry="2" />
<text  x="1058.45" y="751.5" ></text>
</g>
<g >
<title>[[kernel.kallsyms]] (1,887 samples, 0.58%)</title><rect x="194.6" y="437" width="6.8" height="15.0" fill="rgb(228,85,52)" rx="2" ry="2" />
<text  x="197.58" y="447.5" ></text>
</g>
<g >
<title>runtime.systemstack (14,776 samples, 4.53%)</title><rect x="1130.1" y="869" width="53.4" height="15.0" fill="rgb(231,196,23)" rx="2" ry="2" />
<text  x="1133.06" y="879.5" >runti..</text>
</g>
<g >
<title>gvisor.dev/gvisor/pkg/tcpip/stack.(*neighborCache).handleUpperLevelConfirmation (2,502 samples, 0.77%)</title><rect x="1078.1" y="773" width="9.1" height="15.0" fill="rgb(208,221,42)" rx="2" ry="2" />
<text  x="1081.14" y="783.5" ></text>
</g>
</g>
</svg>
