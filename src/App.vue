<template>
  <v-app>
    <v-container id="container">
      <v-layout column justify-start align-center wrap>
        <v-flex text-xs-center>
          <h1 class="display-2 font-weight-bold">
            Graph
          </h1>
        </v-flex>
        <v-layout column justify-start align-center wrap>
          <v-flex xs12>
            <svg id="svg" :width="width" :height="height"
              viewBox="0 0 200 50" version="1.1" xmlns="http://www.w3.org/2000/svg">
              <path :d="d_linear" fill="none" stroke="grey" />
              <path :d="d_smooth" fill="none" stroke="red" />
            </svg>
          </v-flex>
        </v-layout>
      </v-layout>
    </v-container>
  </v-app>
</template>

<script>
/* eslint camelcase: [0] */
/* eslint no-unused-vars: [1] */
import * as R from 'ramda';

const max = 30;
const smoothing = 0.2;

const points = [
  [0, 2], [10, 16], [20, 0], [30, 21], [40, 11], [50, 28], [60, 9], [70, 18], [80, 12], [90, 29],
  [100, 4], [110, 28], [120, 3], [130, 9], [140, 0], [150, 16], [160, 8], [180, 29], [190, 7], [200, 16]
];

const int = Math.trunc;
const print = s => console.log(`[Graph] ${s}`);
const debug = R.tap(R.compose(print, JSON.stringify));

// Turn the function arguments into a list.
const listify = R.unapply(R.identity);

// Find DOM for the key given.
const getElem = R.compose(
  R.when(R.is(Function), R.call),
  R.prop(R.__, {
    body: () => document.body,
    svg: () => document.getElementById('svg')
  })
);

// Properties of DOMRect are not accessible, and must be explicitly made accessible.
const ownify = o => R.reduce(
  (acc, key) => R.assoc(key, o[key], acc),
  {},
  R.compose(Object.keys, Object.getPrototypeOf)(o)
);

// Given a DOM element, returns the size information.
const getSize = R.ifElse(
  R.isNil,
  R.always({}),
  R.compose(
    ownify, // Make all the properties of DOMRect accessible.
    R.invoker(0, 'getBoundingClientRect')
  )
);

// For a key given, returns the size property object.
const getElemSize = R.compose(getSize, getElem);

// Note: The function is curried because we may need to flip arguments.
const setSize = R.curry((key, o) => R.assoc(key, getElemSize(key), o));

const setSvgSize = (ratio = 0.7) => (o = {}) => {
  const accessor = R.flip(R.path)(o);
  // const fullHeight = accessor(['body', 'height']) || 0;
  // const heightUsed = accessor(['svg', 'y']) || 0;
  // const heightAvail = int(fullHeight - heightUsed * 0.9);
  const fullWidth = accessor(['body', 'width']) || 0;
  let width = int(fullWidth * ratio);
  // if (width > heightAvail) {
  //   width = heightAvail;
  // }
  o.svg.width = width;
  o.svg.height = width / 2;
  return o;
};

/**
 * Given 2 points, [[x1, y1], [x2, y2]], calculate "x2 - x1" (or "y2 - y1").
 * For "index": "0" gives you the delta for "x" (likewise "1" for "y").
 * @returns {Function}
 */
const delta = (index = 0) => R.compose(
  R.apply(R.subtract), // subtract(x2, x1)
  R.chain(R.nth(index)), // [[x2, y2], [x1, y1]] ---> [x2, x1] (if "index" is "0")
  R.reverse // [[x1, y1], [x2, y2]] ---> [[x2, y2], [x1, y1]]
);

const deltaX = delta(0);
const deltaY = delta(1);

const angle = ([dx, dy]) => Math.atan2(dy, dx);
const dist = ([dx, dy]) => Math.sqrt(Math.pow(dx, 2) + Math.pow(dy, 2));

/**
 * Given 2 points, calculate "angle" and "dist".
 * @param {Array} p1
 * @param {Array} p2
 * @returns {Object} { angle, dist }
 */
const twoPoints = R.compose(
  R.zipObj(['angle', 'dist']), // [angle, dist] --> { angle, dist }
  R.ap([angle, dist]), // [angle([dx, dy]), dist([dx, dy])] --> [angle, dist]
  R.of,
  R.ap([deltaX, deltaY]), // [deltaX([p1, p2]), deltaY([p1, p2]) --> [dx, dy]
  R.of,
  listify // (p1, p2) ---> [p1, p2]
);

/**
 * Get control point needed to draw a bezier curve.
 * Passing "prev" and "next" to "twoPoints", we get "angles" and "dist".
 * Angle for the control point is the same as that of between "prev" and "next".
 * Distance, we apply an arbituary smooth ratio.
 * @param {number} prev
 * @param {number} curr
 * @param {number} next
 * @param {Boolean} flip
 * @returns {Array} [x, y]
 */
const getControlPoint = (prev, curr, next, flip) => R.compose(
  // (3) Use trigonometry, to get [x, y], which is: [102,46].
  ({ angle, dist }) => ([
    curr[0] + (Math.cos(angle) * dist), // x
    curr[1] + (Math.sin(angle) * dist) // y
  ]),
  // (2) { angle: 0.0831, dist: 60.2079 } ---> { angle: 0.0831, dist: 12.0415 }
  // TODO: "smoothing" belongs to external.
  R.evolve({
    angle: R.add(flip ? Math.PI : 0), // Flip the angle for end control point.
    dist: R.multiply(smoothing) // Arbitrary smooth ratio (0.2) for the distance.
  }),
  // (1) Get "angle" and "dist" between "prev" and "next".
  //  [[60,5], [120,10]] ---> { angle: 0.0831, dist: 60.2079 }
  R.always(twoPoints(prev, next))
)();

// A sequence of points are given as an object: { prev2, prev, curr, next }
// Returns a bezier command.
const bezierCommand = R.compose(
  debug,
  R.join(' '), // (7) ---> "C startX,startY, endX,endY, x,y"
  R.prepend('C'), // (6) ---> ["C", "startX, startY", "endX, endY", "x, y"]
  // (5) ---> ["startX, startY", "endX, endY", "x, y"]
  R.ap([
    // (2) ---> "startX, startY"
    R.compose(
      R.apply(getControlPoint), // f(prev2, prev, curr) ---> [cx, cy]
      R.converge(listify, [R.prop('prev2'), R.prop('prev'), R.prop('curr')]) // [prev2, prev, curr]
    ),
    // (3) ---> "endX, endY"
    R.compose(
      R.apply(getControlPoint), // f(prev2, prev, curr, TRUE) ---> [cx, cy]
      R.append(true), // [prev, curr, next] ---> [prev, curr, next, TRUE]
      R.converge(listify, [R.prop('prev'), R.prop('curr'), R.prop('next')]) // [prev, curr, next]
    ),
    // (4) ---> "x, y"
    R.prop('curr')
  ]),
  R.of // (1) obj ---> [obj] (where "obj" being { prev2, prev, curr, next })
);

const moveToCommand = point => `M ${point[0]},${point[1]}`;
const linearCommand = point => `L ${point[0]},${point[1]}`;

// TODO:
// Any ideas to make it point-free. "R.reduce" needs indexing.
const getDatasets = (command, bezier = false) => (points = []) => (
  points.map(([x, y]) => (
    [x, max - y] // Flip y-axis, so "0" starts at the bottom.
  )).reduce((acc, curr, i, orig) => {
    const prev2 = orig[i - 2] || curr;
    const prev = orig[i - 1];
    const next = orig[i + 1] || curr;
    return acc.concat((i === 0) ? moveToCommand(curr)
      : command(bezier ? {
        prev2,
        prev,
        curr,
        next
      } : curr));
  }, []).join(' ')
);

export default {
  name: 'App',
  data: () => ({
    points,
    info: {
      body: { width: 0, height: 0, x: 0, y: 0 },
      svg: { width: 0, height: 0, x: 0, y: 0 }
    }
  }),
  computed: {
    width () {
      return this.info.svg.width;
    },
    height () {
      return this.info.svg.height;
    },
    d_linear () {
      return getDatasets(linearCommand)(this.points);
    },
    d_smooth () {
      return getDatasets(bezierCommand, true)(this.points);
    }
  },
  methods: {
    reset () {
      this.info = R.compose(
        setSvgSize(0.8),
        R.reduce(R.flip(setSize), R.__, ['body', 'svg'])
      )(this.info);
    }
  },
  mounted () {
    this.reset();
  }
}
</script>
