---
layout: post
title:  "Um olhar sobre PagingAndSortingRepository"
date:   2023-01-21
#last_modified_at: 2023-01-21
categories: [Spring]
tag: [springdata,java,paging,repository]
---
#### TL;DR
O método _findAll_ da _interface_ **PagingAndSortingRepository**, faz uma _query_/busca adicional à sua base de dados.  

####  Serendipidade no _App_ do Pássaro Azul

Esses dias durante um _doom scrolling_ no _twitter_, me deparei com um _post_ interessante sobre o _PagingAndSortingRepository_ do Spring Data, basicamente era um alerta sobre a execução de uma _query_ adicional(um simples _count_) no método _findAll_, e, como isso dependendo do teu cenário, poderia ser um gargalo.

![Post do Simon Martinelli - simas_ch](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z1tri6x1cat0rzumubc5.png)
> Esteja atent@ que o método _findAll_ do _PagingAndSortingRepository_ do Spring Data realizará uma query adicional, se você possue uma consulta lenta, ela levará o dobro do tempo!

#### Vendo para crer
Isso me chamou atenção e fui buscar informações.
Olhei na [documentação](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/PagingAndSortingRepository.html) e não encontrei os detalhes específicos sobre isso.
Resolvi subir um projeto que já tenho preparado para vários tipos de experimentações e rodar esse cenário   

**Big surprise**: realmente, executa a _query_ adicional de _count_   

Conforme o autor do _twit_ responde uma possível contramedida é utilizar _getBy(Pageable pageable)_ porém acaba abrindo mão dos metadados do objeto _Page_.   

#### _Mise en Place_
![mise en place](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tl4y37p59km0dho1dlc6.jpg) Foto de <a href="https://unsplash.com/pt/@rudy_issa?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Rudy Issa</a> na <a href="https://unsplash.com/pt-br/fotografias/KVacTm0QeEA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>   
  
Para verificar esse caso, utilizei no projeto:
- Java 11+
- Spring Boot 3+
- Spring Data
- Spring Web
- Actuator
- Lombok (opcional)
- IntelliJ IDEA Community Edition 2022.3.1 (opcional)
- [Mysql Employee DB Sample](https://dev.mysql.com/doc/employee/en/)     

O repositório já implementado pode ser clonado e/ou verificado [aqui!](https://github.com/felipejsm/employees/tree/feature-modeling)

Interface _TitlesRepository_
```java
package com.nssp.employees.data.repositories;

import com.nssp.employees.data.models.Titles;
import org.springframework.data.domain.Pageable;
import org.springframework.data.repository.PagingAndSortingRepository;

import java.util.List;

public interface TitlesRepository extends PagingAndSortingRepository<Titles, Long> {
    List<Titles> findBy(Pageable pageable);
}

```
Classe principal e de controller
```java
package com.nssp.employees;

import com.nssp.employees.data.models.Titles;
import com.nssp.employees.data.repositories.EmployeesRepository;
import com.nssp.employees.data.repositories.TitlesRepository;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.concurrent.TimeUnit;

@SpringBootApplication
@RestController
@RequestMapping("/")
public class EmployeesApplication {

	private EmployeesRepository repository;
	private TitlesRepository titlesRepository;

	public EmployeesApplication(final EmployeesRepository repository,
								final TitlesRepository titlesRepository) {
		this.repository = repository;
		this.titlesRepository = titlesRepository;
	}

	public static void main(String[] args) {
		SpringApplication.run(EmployeesApplication.class, args);
	}

	@GetMapping("page")
	public Page<Titles> get() {
		long startTime = System.nanoTime();
		var response = this.titlesRepository.findAll(PageRequest.of(1, 10000));
		long endTime = System.nanoTime();
		long duration = TimeUnit.SECONDS.convert( (endTime - startTime), TimeUnit.NANOSECONDS);
		System.out.println("Page time: "+duration);
		return response;
	}

	@GetMapping("list")
	public List<Titles> getList() {
		long startTime = System.nanoTime();
		var response = this.titlesRepository.findBy(PageRequest.of(1, 10000));
		long endTime = System.nanoTime();
		long duration = TimeUnit.SECONDS.convert( (endTime - startTime), TimeUnit.NANOSECONDS);
		System.out.println("List time: "+duration);
		return response;
	}
}
```

**Console da execução(list):**
```sql
Hibernate: select t1_0.emp_no,t1_0.title,t1_0.to_date,t1_0.from_date from titles t1_0 limit ?,?
List time: 209
```
**Console da execução(findAll):**
```sql
Hibernate: select t1_0.emp_no,t1_0.title,t1_0.to_date,t1_0.from_date from titles t1_0 limit ?,?
Hibernate: select count(*) from titles t1_0
Page time: 264
```
Com o _actuator_ liberado temos as seguintes métricas:

**http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/page**
```json
{
  "name": "http.server.requests",
  "baseUnit": "seconds",
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 4
    },
    {
      "statistic": "TOTAL_TIME",
      "value": 2.840016801
    },
    {
      "statistic": "MAX",
      "value": 1.752134
    }
  ],
  "availableTags": [
    {
      "tag": "exception",
      "values": [
        "none"
      ]
    },
    {
      "tag": "method",
      "values": [
        "GET"
      ]
    },
    {
      "tag": "error",
      "values": [
        "none"
      ]
    },
    {
      "tag": "outcome",
      "values": [
        "SUCCESS"
      ]
    },
    {
      "tag": "status",
      "values": [
        "200"
      ]
    }
  ]
}
```

**http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/list**
```json
{
  "name": "http.server.requests",
  "baseUnit": "seconds",
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 3
    },
    {
      "statistic": "TOTAL_TIME",
      "value": 1.4389705
    },
    {
      "statistic": "MAX",
      "value": 0
    }
  ],
  "availableTags": [
    {
      "tag": "exception",
      "values": [
        "none"
      ]
    },
    {
      "tag": "method",
      "values": [
        "GET"
      ]
    },
    {
      "tag": "error",
      "values": [
        "none"
      ]
    },
    {
      "tag": "outcome",
      "values": [
        "SUCCESS"
      ]
    },
    {
      "tag": "status",
      "values": [
        "200"
      ]
    }
  ]
}
```

Temos numa pesquisa simples de 10k de registros uma diferença sensível na casa dos milisegundos

#### Discussão

Qual o melhor? qual o pior?
Depende do teu contexto, do _tradeoff_ que você pode fazer
Pois o que você perde ao não utilizar a paginação são dados como número total de elementos, ordenação, etc.

**Metadados**
```json
"pageable": {
    "sort": {
      "empty": true,
      "sorted": false,
      "unsorted": true
    },
    "offset": 100,
    "pageNumber": 1,
    "pageSize": 100,
    "paged": true,
    "unpaged": false
  },
  "totalPages": 4434,
  "totalElements": 443308,
  "last": false,
  "size": 100,
  "number": 1,
  "sort": {
    "empty": true,
    "sorted": false,
    "unsorted": true
  },
  "numberOfElements": 100,
  "first": false,
  "empty": false
```

Espero que tenha sido interessante essa análise   
E você, vai pesar se vale ou não a pena na tua próxima codificação?

Obrigado.