#  Анализ результатов АА и АВ тестирования
## Краткое описание
Анализируем данные по приложению, объединяющего ленту новостей и ленту сообщений. Для этого используем результаты АА и АВ тестов, которые были проведены ранее.

Стэк:

* JupiterHub
* Clickhouse
* Python
* Статистические тесты

# АА - тест

Убедимся, что система сплитования работает корректно и ключевая метрика не отличается между группами в АА-тесте.
Поставили ограничение по времени выгрузки:  '2023-06-23' по '2023-06-29 и выбрали только 2-ую (экспериментальную) и 3-ью (экспериментальную) группы.
Для проведения теста подсчитали CTR для каждого пользователя и построили график с распределением для информации. На графике видим, что распределение экспериментальных групп практически совпадает.

![2023-08-26_16-22-39](https://github.com/Macharaits/My_project/assets/117433497/5ef9c7cc-abcd-436b-b7ed-65a1962d792a)

Теперь для проверки различий проведем бутстрап.
```python 
# Проведем бутстрап
def  bootstrap_aa(n_bootstrap: int = 10_000,
                  B: int = 500,) -> List:
                  pvalues = []
                  for _ in  range(n_bootstrap):
                  group_2 = df_aa[df_aa['exp_group'] == 2]['ctr'].sample(B, replace=False).tolist()
                  group_3 = df_aa[df_aa['exp_group'] == 3]['ctr'].sample(B, replace=False).tolist()
                  pvalues.append(stats.ttest_ind(group_2, group_3, equal_var=False)[1])
                  return pvalues
```
```python 
sum(np.array(pvalues)<0.05)/10000
```
Вывод: Статистически значимые различия получаем в 4,8% , что соответствует ошибке I рода. Система сплитования на группы работает корректно.

# АВ - тест
Также, как и в АА-тесте выгрузили данные с Ckickhouse. Эксперимент проходил с 2023-06-30 по 2023-07-06 включительно. Для эксперимента были задействованы 2 и 1 группы.

Различия распределений по CTR очень хорошо прослеживается. T-test не даст нам гарантированного результата, поэтому будет использовать разные тесты для оценки.

![2023-08-26_16-31-22](https://github.com/Macharaits/My_project/assets/117433497/cca1b528-b493-4813-b54d-eab58f0d4899)

## T-test
Значение p-value (в данном случае 0.685) является вероятностью получить такие же или более экстремальные различия между группами, какие были наблюдены в эксперименте, при условии, что нулевая гипотеза верна. Нулевая гипотеза в данном случае гласит, что нет статистически значимого различия между средними значениями CTR в двух группах (новый алгоритм не влияет на CTR). Чем выше p-value, тем более вероятно, что нулевая гипотеза верна.
```python
stats.ttest_ind(df[df.exp_group == 1].ctr,df[df.exp_group == 2].ctr, equal_var=False) 

Ttest_indResult(statistic=0.4051491913112757, pvalue=0.685373331140751)
```

## Mann-Witney test
```python 
stats.mannwhitneyu(df[df.exp_group == 1].ctr,
                   df[df.exp_group == 2].ctr,
                   alternative = 'two-sided')

MannwhitneyuResult(statistic=55189913.0, pvalue=4.632205841806026e-45)
```

Таким образом, на основе результатов теста Манна-Уитни, можно сделать вывод, что новый алгоритм рекомендации постов во 2-й группе привел к статистически значимому уменьшению CTR по сравнению с группой 1 (контрольной группой). Это отрицательный результат.

Рассмотрите другие методы:Если у вас есть перекос в данных и много выбросов, тесты на сглаженном CTR, бутстреп или бакетное преобразование могут быть более устойчивыми к таким выбросам и дадут более надежные результаты.Для более детального анализа возьмем бакетное преобразование с Манна-Уитни, т.к у нас есть не равномерное распределение ctr во 2 группе, а  бакетное преобразование позволит нам сгладить этот момент. Посмотрим верно ли гипотеза

![2023-08-26_16-41-15](https://github.com/Macharaits/My_project/assets/117433497/9e3b0cf6-096c-40e8-9dbf-77266109e3f1)
![2023-08-26_16-41-26](https://github.com/Macharaits/My_project/assets/117433497/b8bd0dc4-0ddd-4c2f-9de2-6a8216c5ac09)

```python 
#тест Манна-Уитни видит отличие
stats.mannwhitneyu(df[df.exp_group == 1].ctr9,
                   df[df.exp_group == 2].ctr9,
                   alternative = 'two-sided')
MannwhitneyuResult(statistic=0.0, pvalue=6.7601631082665925e-18)
```
Новый алгоритм не учитывает потребности всех пользователей мы видим, что во 2 группе есть две крайности, можно предположить, что в группе есть разногласия и, например были выбраны пользователи не одинаковой возрастной группы, пола и т.д.. Может быть алгоритм выдавал рандомные посты и рекомендации, которые крайне неэффективны  
Я не рекомендую раскатывать новый алгоритм на всех пользователей. Тест нужно доработать есть много выбросов/нестабильных значений

## Linearized likes
Проанализируем тест между группами 0 и 3, 1 и 2 по метрике линеаризованных лайков.
![2023-08-26_16-54-15](https://github.com/Macharaits/My_project/assets/117433497/279189ec-2739-4440-9db3-314ac029b803)
Нулевая гипотеза не верна средние CTR в группах 0 и 3 отличаются. p-value на линейризовааном ctr уменьшилось.
![2023-08-26_16-54-31](https://github.com/Macharaits/My_project/assets/117433497/b36a6e18-d4c7-4e70-95f6-7af400aab191)
На группах 1 и 2 t-тест на линеаризованном ctr показал различия между группами, и они правда есть. Р-value значительно уменьшилось относительно результата на обычном ctr.
Метрика линеаризованных лайков, несмотря на разное распределение в выборках, хорошо себя показала.

