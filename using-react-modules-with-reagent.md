**React component that takes props and has children:**

```
[(reagent/adapt-react-class FlipMove)
         {:duration 500
          :easing "ease"}
         elements]
```

**Higher Order Component: React functional component that takes another component as an argument and returns another component:**

```
(defn react-dnd-component
  []
  (let [decorator (react-dnd/DragDropContext HTML5Backend)]
    [(reagent/adapt-react-class
       (decorator (reagent/reactify-component top-level-component)))]))
```

**Function-as-child component:**

```
[(reagent/adapt-react-class AutoSizer)
 {}
 (fn [dims]
  (let [dims (js->clj dims :keywordize-keys true)] 
   (reagent/as-element [my-component (:height dims)])))]
```



