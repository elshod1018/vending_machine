create function crypt_password(password character varying) returns character varying
language plpgsql
as
$$
BEGIN
if password is null then
raise exception 'Password can not be null';
end if;
return utils.crypt(password,utils.gen_salt('bf',4));
END
$$;--used for crypting password

create function match_password(password character varying, crypted_password character varying) returns boolean
language plpgsql
as
$$
BEGIN
if password is null then
raise exception 'Password can not be null';
end if;
if crypted_password is null then
raise exception 'Crypted password can not be null';
end if;
return crypted_password = utils.crypt(password, crypted_password);
END
$$;--for matching raw password and crypted password 

create function product_add(product_params character varying, admin_params character varying) returns integer
language plpgsql
as
$$
DECLARE
admin record;
new_id int;
admin_data json;
product_data json;
u_username varchar;
u_password varchar;
product_dto helper.product_add_dto;
p_counts int;
BEGIN
if product_params is null or trim(product_params) = '' or product_params = '{}' then
raise exception 'Product parameters can not be null or empty';
end if;
if admin_params is null or trim(admin_params) = '' or admin_params = '{}' then
raise exception 'Admin parameters can not be null or empty';
end if;

    admin_data := admin_params::json;
    u_username := admin_data ->> 'username';
    u_password := admin_data ->> 'password';

    select *
    into admin
    from public.admins a
    where a.username = u_username
      and helper.match_password(u_password, a.password);
    if not FOUND then
        raise exception 'Bad credentials' using detail = 'Only admins can add product';
    end if;

    product_data := product_params::json;
    product_dto.p_name := product_data ->> 'p_name';
    product_dto.p_price := product_data ->> 'p_price';
    product_dto.p_count := product_data ->> 'p_count';

    if product_dto.p_count<1 then
        raise exception 'Invalid product count to add';
    end if;
    select sum(p.p_count) into p_counts from public.products p where p.p_count > 0;
    if p_counts + product_dto.p_count > 120 then
        raise exception 'No enough space to add product';
    end if;

    insert into public.products(p_name, p_price, p_count)
    VALUES (product_dto.p_name, product_dto.p_price, product_dto.p_count)
    returning id into new_id;

    return new_id;

END
$$; -- Add product if all right or else throw exception

create function product_buy(product_id integer, money numeric) returns boolean
language plpgsql
as
$$
DECLARE
r_product record;
BEGIN
if product_id is null then
raise exception 'Product id can not be null ';
end if;
if money is null then
raise exception 'Money can not be null ';
end if;

    select * into r_product from public.products p where p.id = product_id and p.p_count > 0;
    if not FOUND then
        raise exception 'Can not found product by id % ',product_id;
    end if;
    if r_product.p_price > money then
        raise exception 'No enough money try again';
    end if;
    update public.products set p_count = r_product.p_count - 1 where id = product_id;
    return true;

END
$$; -- For buy product if input parameters are right buy or else throw exception

create procedure show_products()
language plpgsql
as
$$
BEGIN
select string_agg('' || p.id, ',') as product_ids,
p.p_name as product_name,
p_price as price,
sum(p.p_count) as count
from public.products p
where p.p_count > 0
group by p.p_name, p.p_price
order by p.p_name;
END
$$;-- used for show products