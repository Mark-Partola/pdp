- `staleTime` - дефолтное 0, все запросы сразу устаревшие
- `gcTime` - если запрос inactive, время через которое будет вытеснен

- `placeholderData` - данные, которые будут использоваться если нет данных, например, при первом запросе или при последующих запросах.
Принимает значение, либо функцию. Есть встроенная функция `keepPreviousData`

`isPlaceholderData` - флаг, если данные взяты из `placeholderData`, можно использовать для анимации

`isFetching` - на каждый перезапрос, даже в фоне

`useQuery` передает `signal`

`initialData` в отличие от `placeholderData` попадает в кэш. Если нет данных данные `placeholderData` просто показываются.

```ts
type QueryStatus = 'pending' | 'error' | 'success';
type FetchStatus = 'fetching' | 'paused' | 'idle';

status: QueryStatus
fetchStatus: FetchStatus. 
```


`isPending` - есть ли наличие данных в кэше, то есть пока данные не загрузились флаг будет `true`. Даже если запрос не `enabled`, он будет `isPending`. Если завязываться только на него - может возникнуть в UI ситуация с вечным состоянием загрузки.

```ts
if (status === 'pending' && fetchStatus === 'fetching') {
  // показывает лоадер
}
```

---
`isPending` - данных нет
`isFetching` - запрос в процессе
`isLoading` - данных нет, но запрос в процессе - `isPending` && `isFetching`

Бесконечная подгрузка:

![[Screenshot 2024-11-07 at 20.51.38.png]]

Так как `queryKey` идет в паре с `queryFn` - можно вынести отдельно.
Такой подход обеспечивает использование вне компонентов, в отличие от выноса в кастомные хуки.

```ts
export const getTasksListQueryOptions = () => {
	return infinityQueryOptions({
		queryKey: ['tasks', 'list'],
		queryFn: meta => api.getList({ page: meta.pageParam }, meta),
		initialPageParam: 1,
		getNextPageParam: result => result.next,
		select: result => result.pages.flatMap(page => page.data),
	})
}
```

```ts
export const getTasksListQueryOptions = ({ page }: { page: number }) => {
	return queryOptions({
		queryKey: ['tasks', 'list', { page }],
		queryFn: meta => api.getList({ page }, meta),
	})
}
```

----

Ключ в мутациях необязательный. Нужен только если необходимо узнать статус мутации из другого места.

Инвалидацию лучше делать через `onSettled`, в случае если где-то нет транзакционной целостности и что-то все таки поменялось при ошибке.

Можно передавать в `invalidateQueries` результат `queryOptions`

```ts
useMutation({
	mutationFn: api.createTask,
	onSettled: () => {
		queryClient.invalidateQueries(api.getTasksListQueryOptions())
	}
})
```

`isPending` на мутации - от старта до завершения мутации

Может быть ситуация на мутации - она уже закончилась, например, кнопку перестали блокировать, происходит инвалидация и перезапрос кэша - и в этот момент есть задержка от окончания блокировки кнопки до появления обновленных данных.
Можно решить, сделав асинхронным `onSettled` и дождаться обновления кэша:

```ts
useMutation({
	mutationFn: api.createTask,
	onSettled: async () => {
		await queryClient.invalidateQueries(api.getTasksListQueryOptions())
	}
})
```

`mutateAsync` - может кинуть ошибку

-----

В `onSuccess` передается ответ мутации, вторым аргументом параметры с которыми была вызвана мутация.

Можно сделать Pessimistic Update

```ts
const deleteMutation = useMutation({
	mutationFn: api.deleteTask,
	onSettled: () => {
		queryClient.invalidateQueries(api.getTasksListQueryOptions())
	},
	onSuccess: (_, taskId) => {
		const key = api.getTasksListQueryOptions().queryKey
		const tasks = queryClient.getQueryData(key)

		if (tasks) {
			queryClient.setQueryData(key, tasks.filter(it => it.id !== taskId))
		}
	}
})
```

Можно сделать сразу, не получая заранее

```ts
queryClient.setQueryData(
	api.getTasksListQueryOptions().queryKey,
	tasks => tasks?.filter(it => it.id !== taskId),
)
```

```ts
deleteMutation.mutate(item.id)
...

// Можно блокировать кнопки используя этот флаг
deleteMutation.isPending
// Либо блокировать конкретную
deleteMutation.variables === item.id
```

-----

Оптимистичные обновления:

1. На вызов мутации отменяем все исходящие запросы, которые могут конфликтовать с оптимистичным обновлением для избежания состояния гонки
2. Получаем текущее состояние
3. Обновляем состояние
4. Возвращаем полученное состояние из шага 2 из `onMutate` - это значение будет передано как контекст в `onSettled`, `onError` и `onSuccess`
5. на `onError` восстанавливаем предыдущее состояние из контекста
6. Инвалидируем для дальнейшего перезапроса то, что было отменено на шаге 1 и то, что необходимо для отображения результата мутации

```ts
useMutation({
    mutationFn: api.updateTask,
    onMutate: async task => {
      await queryClient.cancelQueries({
        queryKey: [api.baseKey]
      });

      const prev = queryClient.getQueryData(
        api.getTasksListQueryOptions().queryKey
      );

      queryClient.setQueryData(
        api.getTasksListQueryOptions().queryKey,
        tasks =>
          tasks?.map(it =>
            it.id === task.id ? { ...it, ...task } : it
          )
      );

      return { prev };
    },
    onError: (_, __, context) => {
      if (context) {
        queryClient.setQueryData(
          api.getTasksListQueryOptions().queryKey,
          context.prev
        );
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({
        queryKey: [api.baseKey]
      });
    }
  });

```

------


