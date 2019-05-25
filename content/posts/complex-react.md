---
title: "Building a complex web component with Facebook's React Library"
date: 2014-01-03T00:00:00+01:00
---
**Edit (12-2016):** This article has been updated for React 0.14 and ES2015

React looks set to be the hot front end technology of 2014 with [some](https://twitter.com/floydophone/status/418207644919545856) even calling 2014 the Year of React. So I thought I'd introduce it with a tutorial and hopefully learn something myself too. Here's what we'll be building:

<iframe src="http://mtharrison.github.io/react-resistance-calculator/" style="height: 550px; width: 700px; border: none; display:block; margin:auto" ></iframe>

I'm going to show you how to create a complex, interactive web component with [React](facebook.github.io/react/). To this end, I will be creating a 5 band resistance calculator. This component will consist of an SVG diagram of a electronic resistor component with coloured bands, indicators of resistance and tolerance and dropdowns for changing the colours of the 5 bands.

Initially I wanted to use SVG gradients for the resistor fills, but they're not supported by React/JSX (**Update:** I've opened a [pull request](https://github.com/facebook/react/pull/819) to fix this).

Firstly though, let's talk a little about [React](facebook.github.io/react/).

### What is react?

[React](facebook.github.io/react/) from Facebook is:

> A Javascript Library for Creating User Interfaces

When React was introduced to the community, there was mixed feelings, with some accusing Facebook of flying in the face of time-tested and accepted standards. This didn't faze the React team and they actually adopted a snarky tweet as their strapline!

![Image of the finished product](https://s3-eu-west-1.amazonaws.com/mattharrison/cowboy-tweet.png)

I won't give you a complete breakdown of the library, I suggest you read all the info on their page and have a look at the project on [Github](https://github.com/facebook/react).

After assessing the library myself I've found both pros and cons but I think overall the pros outweigh the cons.

Pros:

* Nice declarative style, you just specific how your UI should look without worry about manipulating the DOM to keep it that way
* JSX looks like HTML, so it's intuitive for designers and is quicker to read and visualise
* Once you've got the hang of it, it's quite quick to develop in
* It makes no assumptions about the rest of your stack so you can integrate it with anything
* Components are composable and/or nestable
* There are some big names behind it or using it in production (Facebook, instagram, Khan Academy)
* It's fast because it uses a virtual DOM diff to calculate operations.

Cons:

* If you use the JSX extension, it will probably break a lot of your syntax highlighting in your IDE
* You'll need to find a way to get JSHint working with your code (there's a fork here: https://npmjs.org/package/jsxhint)
* Mixing logic with Markup feels a bit dirty.

### Building the calculator

Now on to the fun stuff. I will be making the calculator by combining several subcomponents together. Below is a diagram of the different components that will make up the calculator:

![Subcomponents](https://s3-eu-west-1.amazonaws.com/mattharrison/resistance-calculator-annotated.png)

Here's how the hierarchy looks in JSX:

<pre><code class="language-markup"> &lt;ResistanceCalculator&gt;
    &lt;OhmageIndicator /&gt;
    &lt;ToleranceIndicator /&gt;
    &lt;SVGResistor  /&gt;
    &lt;BandSelector /&gt;
    &lt;BandSelector /&gt;
    &lt;BandSelector /&gt;
    &lt;BandSelector /&gt;
    &lt;BandSelector /&gt;
    &lt;ResetButton /&gt;
&lt;/ResistanceCalculator&gt;</code></pre>

I'll post the code for each component here with a small discussion of what's going on inside.

#### components/OhmageIndicator.js

This component displays the actual calculated ohmage of the resistor. Notice it doesn't actually do the calculation, a higher level component takes care of that. The only logic in this component is concerned with rendering an appropriate unit of resistance, be it Ω, KΩ or MΩ.

This component only expects 1 `prop` to be passed, `resistance`.

<pre><code class="language-javascript">import React, { Component } from 'react';

const OhmageIndicator = ({ resistance }) => {
    const formatResistance = () => {
        const r = parseFloat(resistance);
        const MILLION = 1000000;
        const THOUSAND = 1000;

        if (r > MILLION)
            return(r / MILLION).toFixed(1) + "MΩ";
        if(r > THOUSAND)
            return (r / THOUSAND).toFixed(1) + "KΩ";
        return r.toFixed(1) + "Ω";
    }
    return (
        &lt;p id="resistor-value"&gt;
            {formatResistance()}
        &lt;/p&gt;
    );
};

OhmageIndicator.propTypes = {
    resistance: React.PropTypes.number.isRequired
};

export default OhmageIndicator</code></pre>

#### components/ToleranceIndicator.js

Similar to the OhmageIndicator, this component isn't very smart, it just takes a numeric value as a `prop` and shows it as a ±% or shows nothing if the tolerance is 0.

The function `printTolerance()` is acting like a view helper in a templating language, taking a data model and prettifying it for ouput.

<pre><code class="language-javascript">import React, { Component } from 'react';

const ToleranceIndicator = ({ tolerance }) => {
    const formatTolerance = () => {
        return tolerance === 0 ?
            "" :
            "±" + tolerance + "%";
    };
    return (
        &lt;p id="tolerance-value"&gt;
            {formatTolerance()}
        &lt;/p&gt;
    );
};

ToleranceIndicator.propTypes = {
    tolerance: React.PropTypes.number.isRequired
};

export default ToleranceIndicator;</code></pre>

#### components/SVGResistor.js

This component dynamically renders an SVG drawing of the resistor. It has a method to translate a band's numeric value to its corresponding colour via an object passed in as a `prop`.

<pre><code class="language-javascript">import React from 'react';

const SVGResistor = ({ bands, bandOptions }) => {
    const bandPositions = [70,100,130,160,210];
    return (
        &lt;svg width={300} height={100} version="1.1" xmlns="http://www.w3.org/2000/svg"&gt;
            &lt;rect x={0} y={26} rx={5} width={300} height={7} fill="#d1d1d1" /&gt;
            &lt;rect x={50} y={0} rx={15} width={200} height={57} fill="#FDF7EB" /&gt;
            {bands.map((b, i) =>  (
                &lt;rect
                    key={i}
                    x={bandPositions[i]}
                    width={7}
                    height={57}
                    fill={bandOptions[b].color}
                /&gt;
            ))}
        </svg>
    );
};

SVGResistor.propTypes = {
    bands: React.PropTypes.array.isRequired,
    bandOptions: React.PropTypes.array.isRequired
};

export default SVGResistor;
</code></pre>

#### components/BandSelector.js

This renders a select menu. To communicate changes back to the owner, the owner has passed a callback as a `prop`, thus creating a line of communication back to the owner. The component will call that callback with some arguments to notify the owner it needs to update its state.

<pre><code class="language-javascript">import React, { Component } from 'react';

class BandSelector extends Component {
    constructor(props) {
        super(props);
    }
    handleChange() {
        this.props.onChange(this.props.band, parseInt(this.refs.menu.value));
    }
    render(){
        const { band, options, value } = this.props;
        return (
            &lt;div className="band-option"&gt;
                &lt;label&gt;Band {band + 1}&lt;/label&gt;
                &lt;select ref="menu" value={value} onChange={::this.handleChange}&gt;
                {options.map((o, i) => (
                    &lt;option key={i} value={i}>{o.label}</option>
                ))}
                &lt;/select&gt;
            &lt;/div&gt;
        );
    }
};

BandSelector.propTypes = {
    onChange: React.PropTypes.func.isRequired,
    band: React.PropTypes.number.isRequired,
    options: React.PropTypes.array.isRequired,
    value: React.PropTypes.number.isRequired
};

export default BandSelector;
</code></pre>

#### components/index.js (ResistanceCalculator)

This component is the glue that holds the calculator together and manages all the `state`, passes `props` and performs the resistance calculation.

When the state in this component changes, and props that have been passed will also update the children's DOM.

<pre><code class="language-javascript">import React, { Component } from 'react';

import BandSelector from './BandSelector';
import OhmageIndicator from './OhmageIndicator';
import ResetButton from './ResetButton';
import SVGResistor from './SVGResistor';
import ToleranceIndicator from './ToleranceIndicator';

class ResistanceCalculator extends Component {
    constructor(props) {
        super(props);
        this.state = {
            bands: props.config.bands.map(v => v.value || 0),
            resistance: 0,
            tolerance: 0
        };
    }
    getMultiplier() {
        if(this.state.bands[3] == 10)
            return 0.1;
        if(this.state.bands[3] == 11)
            return 0.01;
        return Math.pow(10, this.state.bands[3]);

    }
    calculateResistance() {
        return this.getMultiplier() *
            ((100 * this.state.bands[0]) +
            (10  * this.state.bands[1]) +
            (1   * this.state.bands[2]));
    }
    updateBandState(band, value) {
        const { bandOptions } = this.props.config;
        const { bands } = this.state;
        bands[band] = value;
        this.setState({
            bands,
            resistance: this.calculateResistance(),
            tolerance: bandOptions[this.state.bands[4]].tolerance
        });
    }
    reset() {
        this.setState({
            bands: [0,0,0,0,0],
            resistance: 0,
            tolerance: 0
        });
    }
    render() {
        const { bandOptions, bands: bandsConfig } = this.props.config;
        const { bands, resistance, tolerance } = this.state;
        return (
            &lt;div&gt;
                &lt;SVGResistor bands={bands} bandOptions={bandOptions} /&gt;
                &lt;OhmageIndicator resistance={resistance} /&gt;
                &lt;ToleranceIndicator tolerance={tolerance} /&gt;
                {bands.map((b, i) => {
                    const options = bandOptions.filter((b, j) => (
                        bandsConfig[i].omitOptions.indexOf(j) === -1
                    ));
                    return (
                        &lt;BandSelector
                            options={options}
                            key={i}
                            band={i}
                            value={b}
                            onChange={::this.updateBandState}
                        /&gt;
                    );
                })}
                &lt;ResetButton onClick={::this.reset}/&gt;
            &lt;/div&gt;
        );
    }
};

ResistanceCalculator.propTypes = {
    config: React.PropTypes.object.isRequired
};

export default ResistanceCalculator;
</code></pre>

Lastly to instantiate the component and mount it to the DOM, passing in some configuration:

#### main.js

<pre><code class="language-javascript">import React, { Component } from 'react';
import { render } from 'react-dom';

import ResistanceCalculator from './components/ResistanceCalculator';

const config = {
    bandOptions: [
        { tolerance: 0, color: "black", label: "None" },
        { tolerance: 1, color: "brown", label: "Brown"},
        { tolerance: 2, color: "red", label: "Red"},
        { color: "orange", label: "Orange"},
        { color: "yellow", label: "Yellow"},
        { tolerance: 0.5, color: "green", label: "Green"},
        { tolerance: 0.25, color: "blue", label: "Blue"},
        { tolerance: 0.10, color: "violet", label: "Violet"},
        { tolerance: 0.05, color: "grey", label: "Grey"},
        { color: "white", label: "White"},
        { tolerance: 5, color: "#FFD700", label: "Gold"},
        { tolerance: 10, color: "#C0C0C0", label: "Silver"}
    ],
    bands: [
        { omitOptions: [10,11] },
        { omitOptions: [10,11] },
        { omitOptions: [10,11] },
        { omitOptions: [8,9] },
        { omitOptions: [3,4,9] }
    ]
};

render(&lt;ResistanceCalculator config={config} /&gt;, document.getElementById('container'));
</code></pre>

### The result

<iframe src="http://mtharrison.github.io/react-resistance-calculator/" style="height: 550px; width: 700px; border: none; display:block; margin:auto" ></iframe>

Please feel free to comment with suggestions or improvements.
