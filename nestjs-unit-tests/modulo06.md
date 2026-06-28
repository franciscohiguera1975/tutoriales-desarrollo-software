# Tests Unitarios en NestJS — Página 6
## Módulo 6 · Módulo Posts
### Tests completos de PostsService y PostsController

---

## PostsService — createQueryBuilder y paginate

`PostsService.findAll()` usa `createQueryBuilder` con encadenamiento de
métodos (`leftJoinAndSelect`, `where`, `orderBy`) y luego llama a
`paginate`. El queryBuilder se mockea con `mockReturnThis()` en cada
método encadenado y se restaura en `beforeEach` después de
`clearAllMocks`.

`update()` llama a `this.findOne(id)` internamente, que a su vez usa
`this.postsRepository.findOne()`. Al mockear el repositorio, la llamada
interna se intercepta automáticamente.

`remove()` devuelve `boolean` basado en el campo `affected` del
resultado de `delete()`.

---

### `src/posts/posts.service.spec.ts`

```typescript
jest.mock('nestjs-typeorm-paginate', () => ({
  paginate: jest.fn(),
}));

import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { paginate } from 'nestjs-typeorm-paginate';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { Category } from '../categories/category.entity';

const mockPaginate = paginate as jest.Mock;

const POST_ID      = '11111111-1111-1111-1111-111111111111';
const POST_ID_2    = '22222222-2222-2222-2222-222222222222';
const CATEGORY_ID  = '44444444-4444-4444-4444-444444444444';
const NOT_FOUND_ID = '99999999-9999-9999-9999-999999999999';

describe('PostsService', () => {
  let service: PostsService;

  const mockQueryBuilder = {
    leftJoinAndSelect: jest.fn().mockReturnThis(),
    where:             jest.fn().mockReturnThis(),
    orderBy:           jest.fn().mockReturnThis(),
  };

  const mockPostsRepository = {
    create:             jest.fn(),
    save:               jest.fn(),
    findOne:            jest.fn(),
    delete:             jest.fn(),
    createQueryBuilder: jest.fn(),
  };

  const mockCategoriesRepository = {
    findOne: jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();
    mockPostsRepository.createQueryBuilder.mockReturnValue(mockQueryBuilder);
    mockQueryBuilder.leftJoinAndSelect.mockReturnThis();
    mockQueryBuilder.where.mockReturnThis();
    mockQueryBuilder.orderBy.mockReturnThis();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        PostsService,
        { provide: getRepositoryToken(Post),     useValue: mockPostsRepository },
        { provide: getRepositoryToken(Category), useValue: mockCategoriesRepository },
      ],
    }).compile();

    service = module.get<PostsService>(PostsService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('create()', () => {

    it('should return null when category does not exist', async () => {
      mockCategoriesRepository.findOne.mockResolvedValue(null);
      expect(await service.create({ title: 'Post', content: 'Contenido', categoryId: CATEGORY_ID })).toBeNull();
    });

    it('should create and return a post when category exists', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tech' };
      const mockPost     = { id: POST_ID, title: 'Post', content: 'Texto', category: mockCategory };
      mockCategoriesRepository.findOne.mockResolvedValue(mockCategory);
      mockPostsRepository.create.mockReturnValue(mockPost);
      mockPostsRepository.save.mockResolvedValue(mockPost);

      const result = await service.create({ title: 'Post', content: 'Texto', categoryId: CATEGORY_ID });
      expect(result).toEqual(mockPost);
    });

    it('should call categoriesRepository.findOne with correct id', async () => {
      mockCategoriesRepository.findOne.mockResolvedValue(null);
      await service.create({ title: 'Post', content: 'Texto', categoryId: CATEGORY_ID });
      expect(mockCategoriesRepository.findOne).toHaveBeenCalledWith({ where: { id: CATEGORY_ID } });
    });

    it('should return null when repository throws', async () => {
      mockCategoriesRepository.findOne.mockRejectedValue(new Error('DB connection lost'));
      expect(await service.create({ title: 'Post', content: 'Texto', categoryId: CATEGORY_ID })).toBeNull();
    });

  });

  describe('findOne()', () => {

    it('should return a post when it exists', async () => {
      const mockPost = { id: POST_ID, title: 'Post encontrado', category: { name: 'Tech' } };
      mockPostsRepository.findOne.mockResolvedValue(mockPost);
      expect(await service.findOne(POST_ID)).toEqual(mockPost);
    });

    it('should return null when post does not exist', async () => {
      mockPostsRepository.findOne.mockResolvedValue(null);
      expect(await service.findOne(NOT_FOUND_ID)).toBeNull();
    });

    it('should call findOne with id and category relation', async () => {
      mockPostsRepository.findOne.mockResolvedValue(null);
      await service.findOne(POST_ID);
      expect(mockPostsRepository.findOne).toHaveBeenCalledWith({
        where: { id: POST_ID },
        relations: ['category'],
      });
    });

    it('should return null when repository throws', async () => {
      mockPostsRepository.findOne.mockRejectedValue(new Error('Timeout'));
      expect(await service.findOne(POST_ID)).toBeNull();
    });

  });

  describe('update()', () => {

    it('should return null when post does not exist', async () => {
      mockPostsRepository.findOne.mockResolvedValue(null);
      expect(await service.update(NOT_FOUND_ID, { title: 'Nuevo', content: 'Texto', categoryId: CATEGORY_ID })).toBeNull();
    });

    it('should return null when new category does not exist', async () => {
      const mockPost = { id: POST_ID, title: 'Viejo', category: { id: CATEGORY_ID } };
      mockPostsRepository.findOne.mockResolvedValue(mockPost);
      mockCategoriesRepository.findOne.mockResolvedValue(null);
      expect(await service.update(POST_ID, { title: 'Nuevo', content: 'Texto', categoryId: NOT_FOUND_ID })).toBeNull();
    });

    it('should update title and content without changing category', async () => {
      const mockPost  = { id: POST_ID, title: 'Viejo', content: 'Texto viejo' };
      const savedPost = { id: POST_ID, title: 'Nuevo', content: 'Texto nuevo' };
      mockPostsRepository.findOne.mockResolvedValue({ ...mockPost });
      mockPostsRepository.save.mockResolvedValue(savedPost);

      const result = await service.update(POST_ID, { title: 'Nuevo', content: 'Texto nuevo' });
      expect(result).toEqual(savedPost);
      expect(mockCategoriesRepository.findOne).not.toHaveBeenCalled();
    });

    it('should update category when new categoryId is provided', async () => {
      const newCategoryId = '55555555-5555-5555-5555-555555555555';
      const mockPost      = { id: POST_ID, title: 'Post', category: { id: CATEGORY_ID } };
      const newCategory   = { id: newCategoryId, name: 'Nueva' };
      const updatedPost   = { ...mockPost, category: newCategory };
      mockPostsRepository.findOne.mockResolvedValue({ ...mockPost });
      mockCategoriesRepository.findOne.mockResolvedValue(newCategory);
      mockPostsRepository.save.mockResolvedValue(updatedPost);

      const result = await service.update(POST_ID, { title: 'Post', content: 'Contenido', categoryId: newCategoryId });
      expect(result?.category).toEqual(newCategory);
    });

    it('should return null when repository throws', async () => {
      mockPostsRepository.findOne.mockRejectedValue(new Error('DB error'));
      expect(await service.update(POST_ID, { title: 'x', content: 'y' })).toBeNull();
    });

  });

  describe('remove()', () => {

    it('should return true when post is deleted', async () => {
      mockPostsRepository.delete.mockResolvedValue({ affected: 1 });
      expect(await service.remove(POST_ID)).toBe(true);
    });

    it('should return false when post does not exist', async () => {
      mockPostsRepository.delete.mockResolvedValue({ affected: 0 });
      expect(await service.remove(NOT_FOUND_ID)).toBe(false);
    });

    it('should return false when repository throws', async () => {
      mockPostsRepository.delete.mockRejectedValue(new Error('FK constraint'));
      expect(await service.remove(POST_ID)).toBe(false);
    });

  });

  describe('findAll()', () => {

    const mockPaginationResult = {
      items: [
        { id: POST_ID,   title: 'Post A', category: { name: 'Tech' } },
        { id: POST_ID_2, title: 'Post B', category: { name: 'News' } },
      ],
      meta: { itemCount: 2, totalItems: 2, itemsPerPage: 10, totalPages: 1, currentPage: 1 },
    };

    it('should return paginated posts without filters', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      const result = await service.findAll({ page: 1, limit: 10 });
      expect(result).toEqual(mockPaginationResult);
    });

    it('should call createQueryBuilder with "post" alias', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10 });
      expect(mockPostsRepository.createQueryBuilder).toHaveBeenCalledWith('post');
    });

    it('should call leftJoinAndSelect for category relation', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10 });
      expect(mockQueryBuilder.leftJoinAndSelect).toHaveBeenCalledWith('post.category', 'category');
    });

    it('should call where when search is provided', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10, search: 'NestJS' });
      expect(mockQueryBuilder.where).toHaveBeenCalled();
    });

    it('should not call where when search is not provided', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10 });
      expect(mockQueryBuilder.where).not.toHaveBeenCalled();
    });

    it('should call orderBy when sort is provided', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10, sort: 'title', order: 'ASC' });
      expect(mockQueryBuilder.orderBy).toHaveBeenCalledWith('post.title', 'ASC');
    });

    it('should not call orderBy when sort is not provided', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10 });
      expect(mockQueryBuilder.orderBy).not.toHaveBeenCalled();
    });

    it('should pass page and limit to paginate', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 2, limit: 5 });
      expect(mockPaginate).toHaveBeenCalledWith(mockQueryBuilder, { page: 2, limit: 5 });
    });

    it('should return null when paginate throws', async () => {
      mockPaginate.mockRejectedValue(new Error('DB timeout'));
      expect(await service.findAll({ page: 1, limit: 10 })).toBeNull();
    });

  });

});
```

---

## PostsController — NotFoundException e InternalServerErrorException

`PostsController` tiene dos tipos de error según el método:
`create()` lanza `NotFoundException` si la categoría no existe.
`findAll()` lanza `InternalServerErrorException` si hay un fallo
general. `findOne()`, `update()` y `remove()` lanzan `NotFoundException`.

`remove()` devuelve el id del post eliminado como `data`.

---

### `src/posts/posts.controller.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException, InternalServerErrorException } from '@nestjs/common';
import { PostsController } from './posts.controller';
import { PostsService } from './posts.service';

const POST_ID      = '11111111-1111-1111-1111-111111111111';
const CATEGORY_ID  = '44444444-4444-4444-4444-444444444444';
const NOT_FOUND_ID = '99999999-9999-9999-9999-999999999999';

describe('PostsController', () => {
  let controller: PostsController;

  const mockPostsService = {
    create:  jest.fn(),
    findAll: jest.fn(),
    findOne: jest.fn(),
    update:  jest.fn(),
    remove:  jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      controllers: [PostsController],
      providers: [
        { provide: PostsService, useValue: mockPostsService },
      ],
    }).compile();

    controller = module.get<PostsController>(PostsController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  describe('create()', () => {

    it('should return SuccessResponseDto with the created post', async () => {
      const dto      = { title: 'Nuevo', content: 'Contenido', categoryId: CATEGORY_ID };
      const mockPost = { id: POST_ID, ...dto };
      mockPostsService.create.mockResolvedValue(mockPost);

      const result = await controller.create(dto);
      expect(result).toEqual({ success: true, message: 'Post created successfully', data: mockPost });
    });

    it('should throw NotFoundException when service returns null', async () => {
      mockPostsService.create.mockResolvedValue(null);
      await expect(controller.create({ title: 'Post', content: 'Texto', categoryId: NOT_FOUND_ID }))
        .rejects.toThrow(NotFoundException);
    });

    it('should call postsService.create with the provided dto', async () => {
      const dto = { title: 'Post', content: 'Texto', categoryId: CATEGORY_ID };
      mockPostsService.create.mockResolvedValue(null);
      try { await controller.create(dto); } catch {}
      expect(mockPostsService.create).toHaveBeenCalledWith(dto);
    });

  });

  describe('findAll()', () => {

    const mockPagination = {
      items: [{ id: POST_ID, title: 'Post A' }],
      meta: { currentPage: 1, totalPages: 1, itemCount: 1, totalItems: 1, itemsPerPage: 10 },
    };

    it('should return SuccessResponseDto with paginated posts', async () => {
      mockPostsService.findAll.mockResolvedValue(mockPagination);
      const result = await controller.findAll({ page: 1, limit: 10 });
      expect(result.success).toBe(true);
      expect(result.data).toEqual(mockPagination);
    });

    it('should throw InternalServerErrorException when service returns null', async () => {
      mockPostsService.findAll.mockResolvedValue(null);
      await expect(controller.findAll({ page: 1, limit: 10 }))
        .rejects.toThrow(InternalServerErrorException);
    });

    it('should cap limit to 100 when limit exceeds 100', async () => {
      mockPostsService.findAll.mockResolvedValue(mockPagination);
      const query = { page: 1, limit: 200 };
      await controller.findAll(query);
      expect(mockPostsService.findAll).toHaveBeenCalledWith(
        expect.objectContaining({ limit: 100 })
      );
    });

  });

  describe('findOne()', () => {

    it('should return SuccessResponseDto when post exists', async () => {
      const mockPost = { id: POST_ID, title: 'Bienvenidos', content: 'Primer post' };
      mockPostsService.findOne.mockResolvedValue(mockPost);

      const result = await controller.findOne(POST_ID);
      expect(result).toEqual({ success: true, message: 'Post retrieved successfully', data: mockPost });
    });

    it('should throw NotFoundException when post does not exist', async () => {
      mockPostsService.findOne.mockResolvedValue(null);
      await expect(controller.findOne(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

    it('should throw NotFoundException with correct message', async () => {
      mockPostsService.findOne.mockResolvedValue(null);
      await expect(controller.findOne(POST_ID)).rejects.toThrow('Post not found');
    });

    it('should call postsService.findOne with the correct id', async () => {
      mockPostsService.findOne.mockResolvedValue({ id: POST_ID, title: 'Test' });
      await controller.findOne(POST_ID);
      expect(mockPostsService.findOne).toHaveBeenCalledWith(POST_ID);
    });

  });

  describe('update()', () => {

    it('should return SuccessResponseDto with updated post', async () => {
      const mockPost = { id: POST_ID, title: 'Actualizado' };
      mockPostsService.update.mockResolvedValue(mockPost);

      const result = await controller.update(POST_ID, { title: 'Actualizado', content: 'x', categoryId: CATEGORY_ID });
      expect(result).toEqual({ success: true, message: 'Post updated successfully', data: mockPost });
    });

    it('should throw NotFoundException when post or category not found', async () => {
      mockPostsService.update.mockResolvedValue(null);
      await expect(controller.update(NOT_FOUND_ID, { title: 'x', content: 'y' }))
        .rejects.toThrow(NotFoundException);
    });

  });

  describe('remove()', () => {

    it('should return SuccessResponseDto when post is deleted', async () => {
      mockPostsService.remove.mockResolvedValue(true);
      const result = await controller.remove(POST_ID);
      expect(result).toEqual({ success: true, message: 'Post deleted successfully', data: POST_ID });
    });

    it('should throw NotFoundException when post does not exist', async () => {
      mockPostsService.remove.mockResolvedValue(false);
      await expect(controller.remove(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

  });

});
```
