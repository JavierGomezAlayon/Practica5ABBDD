## Ejercicio 6

Haga un análisis del modelo e incluya las restricciones CHECK que considere necesarias.

El modelo está bastante bien implementado pero no hay muchas restricciones check que se apliquen lo cual puede provocar muchos fallos potenciales.
Para mejorar esto voy a intentar incluir las siguientes check que he visto realmente necesarios:

1. Tabla film 
  - rental_duration no puede ser menor a 0
  - replacement_cost no puede ser menor a 0
  - release_year no puede ser mayor del año actual
2. Tabla rental
- rental_date tiene que ser antes o igual que returnal_date
3. Tabla payment
- ammount no pueder 0 o negativo (al parecer ya hay registros de pagos por 0 euros, por lo que lo voy a dejar en que no puede ser negativo)
- payment_date no puede ser después de hoy

## Ejercicio 7
A continuación explicaré que significa esta sentencia:
```sql
last_updated BEFORE UPDATE ON customer 
FOR EACH ROW EXECUTE PROCEDURE last_updated() 
```
Esto lo que hace es que después de cada update, por cada fila se ejecuta un procedimiento (una función) en concreto last_update(), el cual lo que hace es cambiar la variable last_update a la fecha actual. 
Gracicas a esto pueden tener un registro de la fecha en el que se integró cada información de la base de datos, y además saber que fue lo último que se introdujo en la base de datos.

Las demás tablas que utilizan de esta práctica son: actor, address, category, city, country, customer, film, film_actor, film_category, inventory, language, rental, staff y store.

## Ejercicio 10

Las secuencias son un tipo de dato que se creó por el grán número de veces que se necesitaba un tipo de dato como este, en el que gracias a tener un valor por defecto que va cambiando automáticamente consigue el siempre crear valores únicos aunque no se tenga que específicar al añadir una fila en dicha tabla. Esta característica que tanto lo diferencian del resto de tipos hacen que sean ideales para la generación automática de id's. 
Por último queda destacar que también ofrece maneras de configurar este tipo de dato, como por ejemplo haciendo que empieze por un número que tu decidas o que la sucesión sea incrementada en 2 unidades cada vez, dando a este tipo de datos una flexibilidad muy importante para su utilidad en la vida real.

## Resto de ejercicios
El resto de los ejercicios propuestos en esta práctica se han realizado directamente en la aplicación dbeaver y ejecutado en la base de datos dada. Se podrá ver reflejado en el archivo .sql que está subido al repositorio.
Además voy a dejar aquí el script que he realizado en esta práctica para todos los ejercicios que se tratan de realizar código sql:
```sql
# Ejericio 4
# A
create view view_ventas_totales as
select sum(p.amount) as ventas_totales, c.name as categoria
from payment as p 
	inner join rental as r 
		on r.rental_id = p.rental_id 
	inner join inventory as i 
		on i.inventory_id = r.inventory_id 
	inner join film_category as fc
		on fc.film_id = i.film_id 
	inner join category as c
		on c.category_id = fc.category_id 
group by c."name" 
order by ventas_totales desc

select * from view_ventas_totales;


# B Obtenga las ventas totales por tienda, donde se refleje la ciudad, el país (concatenar la ciudad y el país empleando como separador la “,”), y el encargado. Pudiera emplear GROUP BY, ORDER by

create view view_ventas_por_tienda as
select 
    (ci.city || ', ' || co.country) as ciudad_pais,
    CONCAT(m.first_name, ' ', m.last_name) as encargado,
    SUM(p.amount) as ventas_totales
from store s
    inner join address a on s.address_id = a.address_id
    inner join city ci on a.city_id = ci.city_id
    inner join country co on ci.country_id = co.country_id
    inner join staff m on s.manager_staff_id = m.staff_id
    inner join payment p on m.staff_id = p.staff_id
group by ciudad_pais, encargado
order by ventas_totales desc;


# C Obtenga una lista de películas, donde se reflejen el identificador, el título, descripción, categoría, el precio, la duración de la película, clasificación, nombre y apellidos de los actores (puede realizar una concatenación de ambos). Pudiera emplear GROUP BY

create view view_peliculas_y_actores as
select 
    f.film_id as identificador,
    f.title as titulo,
    f.description as descripcion,
    c.name as categoria,
    f.rental_rate as precio,
    f.length as duracion,
    f.rating as clasificacion,
    (a.first_name || ' ' || a.last_name) as actor
from film f
inner join film_category fc on f.film_id = fc.film_id
inner join category c on fc.category_id = c.category_id
inner join film_actor fa on f.film_id = fa.film_id
inner join actor a on fa.actor_id = a.actor_id
group by 
    f.film_id, f.title, f.description, c.name, 
    f.rental_rate, f.length, f.rating, 
    a.first_name, a.last_name
order by f.film_id;

# D

create view view_actores_y_sus_peliculas as
select 
    (a.first_name || ' ' || a.last_name) as actor,
    (c."name"  || ':' || f.title) as categoría_película
from actor a
inner join film_actor fa on a.actor_id = fa.actor_id
inner join film f on fa.film_id = f.film_id
inner join film_category fc on f.film_id = fc.film_id
inner join category c on fc.category_id = c.category_id
group by a.actor_id, c."name", f.title 
order by actor;

# ej 6:

alter table film
add constraint chek_film_rental_duration check (rental_duration >= 0),
add constraint chek_film_replacement_cost check (replacement_cost >= 0),
add constraint chek_film_release_year check (release_year <= extract(year from current_date));

alter table rental
add constraint chk_rental_dates check (rental_date <= return_date);

alter table payment
add constraint chk_payment_amount check (amount >= 0),
add constraint chk_payment_date check (payment_date <= current_date);

# ej 8

create table film_insert_log (
    id serial primary key,
    film_id int not null,
    insert_date timestamp not null
);

create or replace function log_film_insert()
returns trigger as $$
begin
    insert into film_insert_log (film_id, insert_date)
    values (new.film_id, current_timestamp);
    return new;
end;
$$ language plpgsql;

create trigger trg_film_insert
after insert on film
for each row
execute function log_film_insert();

## Comprobación del 8

insert into film (
    title,
    release_year,
    language_id,
    rental_duration,
    rental_rate,
    length,
    replacement_cost,
    rating
)
values (
    'prueba trigger',
    2025,
    1,
    5,
    4.99,
    120,
    19.99,
    'PG'
);

# Ejercicio 9

create table film_delete_log (
    id serial primary key,
    film_id int not null,
    delete_date timestamp not null
);

create or replace function log_film_delete()
returns trigger as $$
begin
    insert into film_delete_log (film_id, delete_date)
    values (old.film_id, current_timestamp);
    return old;
end;
$$ language plpgsql;

create trigger trg_film_delete
after delete on film
for each row
execute function log_film_delete();

## Comprobación del 9

delete from film 
where film_id = 1002;
```
