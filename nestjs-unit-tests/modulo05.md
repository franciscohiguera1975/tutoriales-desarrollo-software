# Tests Unitarios en NestJS — Página 5
## Módulo 5 · Módulo Categories
### Tests completos de CategoriesService y CategoriesController

---

## CategoriesService — paginate y queryBuilder

`CategoriesService` usa `paginate` de `nestjs-typeorm-paginate` en
`findAll()` y encadena métodos de `createQueryBuilder`. Ambos se
mockean a nivel de módulo con `jest.mock()`.

`update()` llama internamente a `this.findOne()`, que a su vez usa
`this.categoryRepo.findOne()`. Al mockear el repositorio, el mock
afecta tanto a la llamada directa como a la indirecta.

`remove()` usa `this.categoryRepo.remove(entity)` (no `delete(id)`),
por lo que el mock del repositorio debe incluir `remove: jest.fn()`.

---

### `src/categories/categories.service.spec.ts`

```typescript
jest.mock('nestjs-typeorm-paginate', () => ({
  paginate: jest.fn(),
}));

import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { paginate } from 'nestjs-typeorm-paginate';
import { CategoriesService } from './categories.service';
import { Category } from './category.entity';

const mockPaginate = paginate as jest.Mock;

const CATEGORY_ID   = '44444444-4444-4444-4444-444444444444';
const CATEGORY_ID_2 = '55555555-5555-5555-5555-555555555555';
const NOT_FOUND_ID  = '99999999-9999-9999-9999-999999999999';

describe('CategoriesService', () => {
  let service: CategoriesService;

  const mockQueryBuilder = {
    where:   jest.fn().mockReturnThis(),
    orderBy: jest.fn().mockReturnThis(),
  };

  const mockCategoryRepository = {
    create:             jest.fn(),
    save:               jest.fn(),
    findOne:            jest.fn(),
    remove:             jest.fn(),
    createQueryBuilder: jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();
    mockCategoryRepository.createQueryBuilder.mockReturnValue(mockQueryBuilder);
    mockQueryBuilder.where.mockReturnThis();
    mockQueryBuilder.orderBy.mockReturnThis();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CategoriesService,
        { provide: getRepositoryToken(Category), useValue: mockCategoryRepository },
      ],
    }).compile();

    service = module.get<CategoriesService>(CategoriesService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('create()', () => {

    it('should create and return a category', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tecnología' };
      mockCategoryRepository.create.mockReturnValue(mockCategory);
      mockCategoryRepository.save.mockResolvedValue(mockCategory);
      expect(await service.create({ name: 'Tecnología' })).toEqual(mockCategory);
    });

    it('should call repository.create with the provided dto', async () => {
      const dto = { name: 'Ciencia' };
      mockCategoryRepository.create.mockReturnValue({});
      mockCategoryRepository.save.mockResolvedValue({ id: CATEGORY_ID_2, ...dto });
      await service.create(dto);
      expect(mockCategoryRepository.create).toHaveBeenCalledWith(dto);
    });

    it('should return null when repository throws', async () => {
      mockCategoryRepository.create.mockReturnValue({});
      mockCategoryRepository.save.mockRejectedValue(new Error('Constraint error'));
      expect(await service.create({ name: 'Duplicado' })).toBeNull();
    });

  });

  describe('findOne()', () => {

    it('should return a category when it exists', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tecnología' };
      mockCategoryRepository.findOne.mockResolvedValue(mockCategory);
      const result = await service.findOne(CATEGORY_ID);
      expect(result).toEqual(mockCategory);
      expect(mockCategoryRepository.findOne).toHaveBeenCalledWith({ where: { id: CATEGORY_ID } });
    });

    it('should return null when category does not exist', async () => {
      mockCategoryRepository.findOne.mockResolvedValue(null);
      expect(await service.findOne(NOT_FOUND_ID)).toBeNull();
    });

    it('should return null when repository throws', async () => {
      mockCategoryRepository.findOne.mockRejectedValue(new Error('DB error'));
      expect(await service.findOne(CATEGORY_ID)).toBeNull();
    });

  });

  describe('update()', () => {

    it('should return null when category does not exist', async () => {
      mockCategoryRepository.findOne.mockResolvedValue(null);
      expect(await service.update(NOT_FOUND_ID, { name: 'Nuevo' })).toBeNull();
    });

    it('should update and return the category', async () => {
      mockCategoryRepository.findOne.mockResolvedValue({ id: CATEGORY_ID, name: 'Viejo' });
      mockCategoryRepository.save.mockResolvedValue({ id: CATEGORY_ID, name: 'Nuevo' });
      expect(await service.update(CATEGORY_ID, { name: 'Nuevo' })).toEqual({ id: CATEGORY_ID, name: 'Nuevo' });
    });

    it('should apply dto fields to the existing category', async () => {
      mockCategoryRepository.findOne.mockResolvedValue({ id: CATEGORY_ID, name: 'Viejo' });
      mockCategoryRepository.save.mockImplementation((cat) => Promise.resolve(cat));
      const result = await service.update(CATEGORY_ID, { name: 'Actualizado' });
      expect(result).toHaveProperty('name', 'Actualizado');
    });

    it('should return null when save throws', async () => {
      mockCategoryRepository.findOne.mockResolvedValue({ id: CATEGORY_ID, name: 'Cat' });
      mockCategoryRepository.save.mockRejectedValue(new Error('Save error'));
      expect(await service.update(CATEGORY_ID, { name: 'x' })).toBeNull();
    });

  });

  describe('remove()', () => {

    it('should return null when category does not exist', async () => {
      mockCategoryRepository.findOne.mockResolvedValue(null);
      expect(await service.remove(NOT_FOUND_ID)).toBeNull();
    });

    it('should call repository.remove with the found category', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tech' };
      mockCategoryRepository.findOne.mockResolvedValue(mockCategory);
      mockCategoryRepository.remove.mockResolvedValue(mockCategory);
      await service.remove(CATEGORY_ID);
      expect(mockCategoryRepository.remove).toHaveBeenCalledWith(mockCategory);
    });

    it('should return the removed category', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tech' };
      mockCategoryRepository.findOne.mockResolvedValue(mockCategory);
      mockCategoryRepository.remove.mockResolvedValue(mockCategory);
      expect(await service.remove(CATEGORY_ID)).toEqual(mockCategory);
    });

    it('should return null when remove throws', async () => {
      mockCategoryRepository.findOne.mockResolvedValue({ id: CATEGORY_ID });
      mockCategoryRepository.remove.mockRejectedValue(new Error('FK constraint'));
      expect(await service.remove(CATEGORY_ID)).toBeNull();
    });

  });

  describe('findAll()', () => {

    const mockPaginationResult = {
      items: [{ id: CATEGORY_ID, name: 'Tech' }, { id: CATEGORY_ID_2, name: 'Ciencia' }],
      meta: { currentPage: 1, totalPages: 1, itemCount: 2, totalItems: 2, itemsPerPage: 10 },
    };

    it('should return paginated categories', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      expect(await service.findAll({ page: 1, limit: 10 })).toEqual(mockPaginationResult);
    });

    it('should call createQueryBuilder with "category" alias', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10 });
      expect(mockCategoryRepository.createQueryBuilder).toHaveBeenCalledWith('category');
    });

    it('should call where when search is provided', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10, search: 'Tech' });
      expect(mockQueryBuilder.where).toHaveBeenCalled();
    });

    it('should not call where when search is not provided', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10 });
      expect(mockQueryBuilder.where).not.toHaveBeenCalled();
    });

    it('should call orderBy when sort is provided', async () => {
      mockPaginate.mockResolvedValue(mockPaginationResult);
      await service.findAll({ page: 1, limit: 10, sort: 'name', order: 'DESC' });
      expect(mockQueryBuilder.orderBy).toHaveBeenCalledWith('category.name', 'DESC');
    });

    it('should return null when paginate throws', async () => {
      mockPaginate.mockRejectedValue(new Error('DB error'));
      expect(await service.findAll({ page: 1, limit: 10 })).toBeNull();
    });

  });

});
```

---

## CategoriesController — InternalServerErrorException vs NotFoundException

`CategoriesController` diferencia dos tipos de error:
`create()` y `findAll()` lanzan `InternalServerErrorException` porque
un fallo en esas operaciones indica un problema del sistema.
`findOne()`, `update()` y `remove()` lanzan `NotFoundException` porque
la ausencia de una categoría es un error de negocio esperado.

`findAll()` también limita el `limit` a 100 como máximo.

---

### `src/categories/categories.controller.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException, InternalServerErrorException } from '@nestjs/common';
import { CategoriesController } from './categories.controller';
import { CategoriesService } from './categories.service';

const CATEGORY_ID  = '44444444-4444-4444-4444-444444444444';
const NOT_FOUND_ID = '99999999-9999-9999-9999-999999999999';

describe('CategoriesController', () => {
  let controller: CategoriesController;

  const mockCategoriesService = {
    create:  jest.fn(),
    findAll: jest.fn(),
    findOne: jest.fn(),
    update:  jest.fn(),
    remove:  jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      controllers: [CategoriesController],
      providers: [
        { provide: CategoriesService, useValue: mockCategoriesService },
      ],
    }).compile();

    controller = module.get<CategoriesController>(CategoriesController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  describe('create()', () => {

    it('should return SuccessResponseDto with created category', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tecnología' };
      mockCategoriesService.create.mockResolvedValue(mockCategory);

      const result = await controller.create({ name: 'Tecnología' });
      expect(result).toEqual({ success: true, message: 'Category created successfully', data: mockCategory });
    });

    it('should throw InternalServerErrorException when service returns null', async () => {
      mockCategoriesService.create.mockResolvedValue(null);
      await expect(controller.create({ name: 'x' })).rejects.toThrow(InternalServerErrorException);
    });

  });

  describe('findAll()', () => {

    const mockPagination = {
      items: [{ id: CATEGORY_ID, name: 'Tech' }],
      meta: { currentPage: 1, totalPages: 1, itemCount: 1, totalItems: 1, itemsPerPage: 10 },
    };

    it('should return SuccessResponseDto with paginated categories', async () => {
      mockCategoriesService.findAll.mockResolvedValue(mockPagination);
      const result = await controller.findAll({ page: 1, limit: 10 });
      expect(result.data).toEqual(mockPagination);
      expect(result.success).toBe(true);
    });

    it('should throw InternalServerErrorException when service returns null', async () => {
      mockCategoriesService.findAll.mockResolvedValue(null);
      await expect(controller.findAll({ page: 1, limit: 10 })).rejects.toThrow(InternalServerErrorException);
    });

    it('should cap limit to 100 when limit exceeds 100', async () => {
      mockCategoriesService.findAll.mockResolvedValue(mockPagination);
      const query = { page: 1, limit: 200 };
      await controller.findAll(query);
      expect(mockCategoriesService.findAll).toHaveBeenCalledWith(
        expect.objectContaining({ limit: 100 })
      );
    });

  });

  describe('findOne()', () => {

    it('should return SuccessResponseDto with category', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tech' };
      mockCategoriesService.findOne.mockResolvedValue(mockCategory);

      const result = await controller.findOne(CATEGORY_ID);
      expect(result).toEqual({ success: true, message: 'Category retrieved successfully', data: mockCategory });
    });

    it('should throw NotFoundException when category does not exist', async () => {
      mockCategoriesService.findOne.mockResolvedValue(null);
      await expect(controller.findOne(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

  });

  describe('update()', () => {

    it('should return SuccessResponseDto with updated category', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tech v2' };
      mockCategoriesService.update.mockResolvedValue(mockCategory);

      const result = await controller.update(CATEGORY_ID, { name: 'Tech v2' });
      expect(result).toEqual({ success: true, message: 'Category updated successfully', data: mockCategory });
    });

    it('should throw NotFoundException when category does not exist', async () => {
      mockCategoriesService.update.mockResolvedValue(null);
      await expect(controller.update(NOT_FOUND_ID, { name: 'x' }))
        .rejects.toThrow(NotFoundException);
    });

    it('should call service.update with the correct id and dto', async () => {
      mockCategoriesService.update.mockResolvedValue({ id: CATEGORY_ID, name: 'Nuevo' });
      const dto = { name: 'Nuevo' };
      await controller.update(CATEGORY_ID, dto);
      expect(mockCategoriesService.update).toHaveBeenCalledWith(CATEGORY_ID, dto);
    });

  });

  describe('remove()', () => {

    it('should return SuccessResponseDto with deleted category', async () => {
      const mockCategory = { id: CATEGORY_ID, name: 'Tech' };
      mockCategoriesService.remove.mockResolvedValue(mockCategory);

      const result = await controller.remove(CATEGORY_ID);
      expect(result).toEqual({ success: true, message: 'Category deleted successfully', data: mockCategory });
    });

    it('should throw NotFoundException when category does not exist', async () => {
      mockCategoriesService.remove.mockResolvedValue(null);
      await expect(controller.remove(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

    it('should call service.remove with the correct id', async () => {
      mockCategoriesService.remove.mockResolvedValue({ id: CATEGORY_ID });
      await controller.remove(CATEGORY_ID);
      expect(mockCategoriesService.remove).toHaveBeenCalledWith(CATEGORY_ID);
    });

  });

});
```
