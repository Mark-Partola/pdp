- Outlet
- index для Route
- Относительные пути
- Не всегда стоит выносить в отдельную группу, если есть один префикс пути. Роуты могут сильно отличаться. Например /list и /list/:id - могут быть совершенно разными

Но можно использовать как группировку, без лэйауты

```jsx
<Route path="list">
	<Route index element={<List />} />
	<Route path=":id" element={<ListItem />} />
</Route>
```

У NavLink есть параметр `end` - для точного совпадения пути при применении стилей

