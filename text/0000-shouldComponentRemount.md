- Start Date: 2018-10-17
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

| Date | Change |
| ---- | ------ |
| 2018-10-17 | Initial |
| 2019-02-03 | Add HOC option |

# Summary

This is a proposal to add a new lifecycle method or higher-order component to handle resetting component state when props change. (Think of this as a controlled key).

Returning true from this method would be equivalent to changing the key prop -
it unmounts and remounts the component instead of preserving the instance.

This method would help component authors define the relationship between components and the data they represent by providing a way to say "the entity this component represents has changed, remount".

Per @vcarl:

> adding a key is a very elegant solution for when you need to re-run mount behaviors, it has the significant drawback of being completely outside the component's control.

This method would also remove a lot of housekeeping boilerplate that is written to make components able to receive props that require a full state reset (typically a combination of componentDidMount, componentWillReceiveProps/getDerivedStateFromProps, and componentDidUpdate).

Userspace implementation of a proposal using a higher-order component: alexkrolick/react-remount-component

# Basic example

## HOC for Function Components/Hooks

(Also works with class components but a lifecycle method may be more natural, see below).

```jsx
function UserCard(props) {
  const { user } = props;
  const [isExpanded, setExpanded] = React.useState(false);
  return (
    <div className="user-card">
      <div>Name: {user.name}</div>
      <div>
        <button onClick={() => setExpanded({ expanded: true })}>
          See Details
        </button>
      </div>
      {isExpanded && (
        <React.Fragment>
          <div>Birthday: {user.birthDate}</div>
          <div>Favorite Color: {user.favoriteColor}</div>
        </React.Fragment>
      )}
    </div>
  );
}

export default withRemount(UserCard, (props, prevProps) => {
  const prevUserId = prevProps.user && prevProps.user.id;
  const nextUserId = props.user && props.user.id;

  if (nextUserId !== prevUserId) {
    return true; // remount!
  }

  return false; // don't remount otherwise
});
```

## Lifecycle Method For Class Components

```diff
class ImgZoomer extends Component<{ imgUrl: string }> {
  constructor(props) {
    super(props);
    this.state = {
      zoom: 1,
      data: null,
      loading: true,
      error: null,
    };
  }
  
  componentDidMount() {
    fetch(this.props.imgUrl).then(
      data => this.setState({data, loading: false}),
      error => this.setState({error, loading: false})
    );
  }
    
+  shouldComponentRemount(newProps) {
+    if (newProps.imgUrl !== this.props.imgUrl) return true;
+  }

  zoomIn = () => this.setState(state => ({ zoom: state.zoom + 1 }));
  
  zoomOut = () => this.setState(state => ({ zoom: state.zoom - 1 }));

  render() {
    if (this.props.error) return 'Error';
    if (this.props.loading) return 'Loading...';
    return (
      <div>
        <ImgComponent scale={(this.state.zoom * 100) + '%'} data={this.state.data} />
        <button onClick={this.zoomIn}>+</button>
        <button onClick={this.zoomOut}>-</button>
      </div>
    );
  }
}
```

# Motivation

The goal is to provide a way for components to manage the relationship between entitites provided by props and
the UI state associated with them.

Example: The [You Might Not Need Derived State][ymnnds-key] blog post recommends a pattern
using an uncontrolled, stateful component with a `key` prop to reset state.

## Downsides of Keys

- `key` is outside the author's control. The consumer of the component is responsible for picking a string for 
  the key that can correctly  represent the reset condition, unless the author of the component
  writes lifecycle code to handle updates without remounting, similar to ["alternative 1"][ymnnds-alt].
- `key` must be serializable to a string.
  - Conditions that rely on several parameters are difficult to represent as strings
  - Non-primitive objects such as instances of classes from third-party libraries may have object equality comparisons implemented, but not necessarily a unique serialization. Values such as floating-point numbers are also difficult to serialize accurately.


# Detailed design

## HOC

Userspace implementation that generates a random key when reset is requested:
https://github.com/alexkrolick/react-remount-component

## Lifecycle method

(TBD if this is feasible)

- Lifecycle order:
  - Not called on initial mount.
  - Called before `shouldComponentUpdate` otherwise.

# Drawbacks

<!--

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.
-->

- There would now be two ways to reset components, `key` and `withRemount`.
- The precedence of these options has to be well-defined.

# Alternatives

- Instead of returning true/false, the method could return a string which would
take the place of the `key` prop.
- The `key` prop could take a function as well as a string.

# Adoption strategy

<!--

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

-->

- Not a breaking change

# How we teach this

<!--

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

-->

- Document usage of HOC or lifecycle method as the "controlled" version of `key` in the docs

# Unresolved questions

<!--

Optional, but suggested for first drafts. What parts of the design are still
TBD?

-->

Should this be a part of core or userspace?

[ymnnds-key]: https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#recommendation-fully-uncontrolled-component-with-a-key
[ymnnds-alt]: https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html#alternative-1-reset-uncontrolled-component-with-an-id-prop
