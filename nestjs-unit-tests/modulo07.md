# Tests Unitarios en NestJS — Página 7
## Módulo 7 · Módulo Cursos
### Tests completos de CursosService y CursosController

---

## CursosService — Mongoose con find/findById/save/deleteOne

A diferencia de los módulos anteriores, `CursosService` usa
**Mongoose** en lugar de TypeORM. Los puntos clave del mock son:

- El modelo se usa como constructor: `new this.cursoModel(data)`.
  El mock debe ser una función (`jest.fn()`) que retorne una instancia
  con `.save()` y `.deleteOne()`.
- `findAll()` encadena `find().skip().limit().populate()`. Cada
  eslabón necesita su propio mock con `mockReturnThis()`.
- `findById()` encadena `.populate()`. Se mockea como objeto separado.
- El servicio inyecta **dos modelos**: `Curso` y `Contenido`.
  El TestingModule debe proveer ambos con `getModelToken`.

---

### `src/cursos/cursos.service.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { getModelToken } from '@nestjs/mongoose';
import { CursosService } from './cursos.service';
import { Curso } from './schemas/curso.schema';
import { Contenido } from './schemas/contenido.schema';

const CURSO_ID     = 'aaaaaaaaaaaaaaaaaaaaaaaa';
const CURSO_ID_2   = 'bbbbbbbbbbbbbbbbbbbbbbbb';
const NOT_FOUND_ID = '999999999999999999999999';

describe('CursosService', () => {
  let service: CursosService;

  // ── cadena para find().skip().limit().populate()
  const mockFindChain = {
    skip:    jest.fn().mockReturnThis(),
    limit:   jest.fn().mockReturnThis(),
    populate: jest.fn(),
  };

  // ── cadena para findById().populate()
  const mockFindByIdChain = { populate: jest.fn() };

  // ── instancia de documento devuelta por new cursoModel()
  const mockCursoInstance = {
    save:      jest.fn(),
    deleteOne: jest.fn(),
    contenidos: [],
  };

  // ── instancia de documento devuelta por new contenidoModel()
  const mockContenidoInstance = { save: jest.fn(), _id: 'cccccccccccccccccccccccc' };

  // ── modelo como función constructora + métodos estáticos
  const mockCursoModel = Object.assign(jest.fn().mockReturnValue(mockCursoInstance), {
    find:    jest.fn().mockReturnValue(mockFindChain),
    findById: jest.fn().mockReturnValue(mockFindByIdChain),
  });

  const mockContenidoModel = Object.assign(jest.fn().mockReturnValue(mockContenidoInstance), {
    create: jest.fn(),
  });

  beforeEach(async () => {
    jest.clearAllMocks();

    // Restaurar implementaciones de cadena tras clearAllMocks
    mockCursoModel.mockReturnValue(mockCursoInstance);
    mockCursoModel.find.mockReturnValue(mockFindChain);
    mockFindChain.skip.mockReturnThis();
    mockFindChain.limit.mockReturnThis();
    mockCursoModel.findById.mockReturnValue(mockFindByIdChain);

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CursosService,
        { provide: getModelToken(Curso.name),     useValue: mockCursoModel },
        { provide: getModelToken(Contenido.name), useValue: mockContenidoModel },
      ],
    }).compile();

    service = module.get<CursosService>(CursosService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  // ─────────────────────────────────────────────────────────────
  describe('create()', () => {

    const dto = {
      nombre: 'NestJS Avanzado', descripcion: 'Aprende NestJS', categoria: 'Backend',
      fecha_inicio: new Date(), fecha_fin: new Date(), nivel: 'intermedio',
      requisitos: ['TypeScript'], precio: 50,
      instructor: { nombre: 'Ana García', email: 'ana@mail.com' },
    };

    it('should create and return a curso', async () => {
      const mockCurso = { _id: CURSO_ID, ...dto };
      mockCursoInstance.save.mockResolvedValue(mockCurso);
      expect(await service.create(dto)).toEqual(mockCurso);
    });

    it('should call the cursoModel constructor with course data', async () => {
      mockCursoInstance.save.mockResolvedValue({ _id: CURSO_ID });
      await service.create(dto);
      expect(mockCursoModel).toHaveBeenCalledWith(expect.objectContaining({ nombre: dto.nombre }));
    });

    it('should return null when save throws', async () => {
      mockCursoInstance.save.mockRejectedValue(new Error('DB error'));
      expect(await service.create(dto)).toBeNull();
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('findAll()', () => {

    const mockCursos = [
      { _id: CURSO_ID,   nombre: 'NestJS Avanzado' },
      { _id: CURSO_ID_2, nombre: 'Docker Básico' },
    ];

    it('should return items with page and limit', async () => {
      mockFindChain.populate.mockResolvedValue(mockCursos);
      const result = await service.findAll({ page: 1, limit: 10 });
      expect(result).toEqual({ items: mockCursos, page: 1, limit: 10 });
    });

    it('should call find().skip().limit().populate()', async () => {
      mockFindChain.populate.mockResolvedValue([]);
      await service.findAll({ page: 1, limit: 10 });
      expect(mockCursoModel.find).toHaveBeenCalled();
      expect(mockFindChain.skip).toHaveBeenCalledWith(0);
      expect(mockFindChain.limit).toHaveBeenCalledWith(10);
      expect(mockFindChain.populate).toHaveBeenCalledWith('contenidos');
    });

    it('should calculate skip correctly for page 2', async () => {
      mockFindChain.populate.mockResolvedValue([]);
      await service.findAll({ page: 2, limit: 5 });
      expect(mockFindChain.skip).toHaveBeenCalledWith(5);
    });

    it('should return null when find throws', async () => {
      mockFindChain.populate.mockRejectedValue(new Error('DB timeout'));
      expect(await service.findAll({ page: 1, limit: 10 })).toBeNull();
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('findOne()', () => {

    it('should return a curso when it exists', async () => {
      const mockCurso = { _id: CURSO_ID, nombre: 'NestJS Avanzado' };
      mockFindByIdChain.populate.mockResolvedValue(mockCurso);
      expect(await service.findOne(CURSO_ID)).toEqual(mockCurso);
      expect(mockCursoModel.findById).toHaveBeenCalledWith(CURSO_ID);
    });

    it('should return null when curso does not exist', async () => {
      mockFindByIdChain.populate.mockResolvedValue(null);
      expect(await service.findOne(NOT_FOUND_ID)).toBeNull();
    });

    it('should return null when findById throws', async () => {
      mockFindByIdChain.populate.mockRejectedValue(new Error('Invalid ObjectId'));
      expect(await service.findOne(CURSO_ID)).toBeNull();
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('update()', () => {

    it('should return null when curso does not exist', async () => {
      mockFindByIdChain.populate.mockResolvedValue(null);
      expect(await service.update(NOT_FOUND_ID, {} as any)).toBeNull();
    });

    it('should update and return the curso', async () => {
      const curso  = { _id: CURSO_ID, nombre: 'Viejo', save: mockCursoInstance.save, contenidos: [] };
      const saved  = { _id: CURSO_ID, nombre: 'Nuevo' };
      mockFindByIdChain.populate.mockResolvedValue(curso);
      mockCursoInstance.save.mockResolvedValue(saved);
      expect(await service.update(CURSO_ID, { nombre: 'Nuevo' } as any)).toEqual(saved);
    });

    it('should return null when save throws', async () => {
      const curso = { _id: CURSO_ID, save: mockCursoInstance.save, contenidos: [] };
      mockFindByIdChain.populate.mockResolvedValue(curso);
      mockCursoInstance.save.mockRejectedValue(new Error('Save error'));
      expect(await service.update(CURSO_ID, {} as any)).toBeNull();
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('remove()', () => {

    it('should return null when curso does not exist', async () => {
      mockFindByIdChain.populate.mockResolvedValue(null);
      expect(await service.remove(NOT_FOUND_ID)).toBeNull();
    });

    it('should call deleteOne on the found curso', async () => {
      const curso = { _id: CURSO_ID, deleteOne: mockCursoInstance.deleteOne };
      mockFindByIdChain.populate.mockResolvedValue(curso);
      mockCursoInstance.deleteOne.mockResolvedValue(curso);
      await service.remove(CURSO_ID);
      expect(mockCursoInstance.deleteOne).toHaveBeenCalled();
    });

    it('should return the deleted curso', async () => {
      const curso = { _id: CURSO_ID, nombre: 'NestJS', deleteOne: mockCursoInstance.deleteOne };
      mockFindByIdChain.populate.mockResolvedValue(curso);
      mockCursoInstance.deleteOne.mockResolvedValue(curso);
      expect(await service.remove(CURSO_ID)).toEqual(curso);
    });

    it('should return null when deleteOne throws', async () => {
      const curso = { _id: CURSO_ID, deleteOne: mockCursoInstance.deleteOne };
      mockFindByIdChain.populate.mockResolvedValue(curso);
      mockCursoInstance.deleteOne.mockRejectedValue(new Error('Delete error'));
      expect(await service.remove(CURSO_ID)).toBeNull();
    });

  });

});
```

---

## CursosController — parámetros separados y mensajes en inglés

El controlador recibe `page` y `limit` como `@Query` individuales
(no como objeto), por lo que en los tests se pasan como argumentos
separados: `controller.findAll(1, 10)`.

Los mensajes de respuesta están en inglés y usan el prefijo `Course`:
`'Course created successfully'`, `'Courses retrieved successfully'`,
`'Course retrieved successfully'`, etc.

---

### `src/cursos/cursos.controller.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException, InternalServerErrorException } from '@nestjs/common';
import { CursosController } from './cursos.controller';
import { CursosService } from './cursos.service';

const CURSO_ID     = 'aaaaaaaaaaaaaaaaaaaaaaaa';
const NOT_FOUND_ID = '999999999999999999999999';

describe('CursosController', () => {
  let controller: CursosController;

  const mockCursosService = {
    create:  jest.fn(),
    findAll: jest.fn(),
    findOne: jest.fn(),
    update:  jest.fn(),
    remove:  jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      controllers: [CursosController],
      providers: [
        { provide: CursosService, useValue: mockCursosService },
      ],
    }).compile();

    controller = module.get<CursosController>(CursosController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  // ─────────────────────────────────────────────────────────────
  describe('create()', () => {

    it('should return SuccessResponseDto with created curso', async () => {
      const mockCurso = { _id: CURSO_ID, nombre: 'NestJS Avanzado' };
      mockCursosService.create.mockResolvedValue(mockCurso);

      const result = await controller.create({} as any);
      expect(result).toEqual({ success: true, message: 'Course created successfully', data: mockCurso });
    });

    it('should throw InternalServerErrorException when service returns null', async () => {
      mockCursosService.create.mockResolvedValue(null);
      await expect(controller.create({} as any))
        .rejects.toThrow(InternalServerErrorException);
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('findAll()', () => {

    const mockResult = {
      items: [{ _id: CURSO_ID, nombre: 'NestJS Avanzado' }],
      page: 1, limit: 10,
    };

    it('should return SuccessResponseDto with courses', async () => {
      mockCursosService.findAll.mockResolvedValue(mockResult);
      const result = await controller.findAll(1, 10);
      expect(result).toEqual({ success: true, message: 'Courses retrieved successfully', data: mockResult });
    });

    it('should call service.findAll with page and limit as object', async () => {
      mockCursosService.findAll.mockResolvedValue(mockResult);
      await controller.findAll(2, 5);
      expect(mockCursosService.findAll).toHaveBeenCalledWith({ page: 2, limit: 5 });
    });

    it('should throw InternalServerErrorException when service returns null', async () => {
      mockCursosService.findAll.mockResolvedValue(null);
      await expect(controller.findAll(1, 10))
        .rejects.toThrow(InternalServerErrorException);
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('findOne()', () => {

    it('should return SuccessResponseDto with curso', async () => {
      const mockCurso = { _id: CURSO_ID, nombre: 'NestJS Avanzado' };
      mockCursosService.findOne.mockResolvedValue(mockCurso);

      const result = await controller.findOne(CURSO_ID);
      expect(result).toEqual({ success: true, message: 'Course retrieved successfully', data: mockCurso });
    });

    it('should throw NotFoundException when curso does not exist', async () => {
      mockCursosService.findOne.mockResolvedValue(null);
      await expect(controller.findOne(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

    it('should throw NotFoundException with correct message', async () => {
      mockCursosService.findOne.mockResolvedValue(null);
      await expect(controller.findOne(NOT_FOUND_ID)).rejects.toThrow('Course not found');
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('update()', () => {

    it('should return SuccessResponseDto with updated curso', async () => {
      const mockCurso = { _id: CURSO_ID, nombre: 'NestJS v2' };
      mockCursosService.update.mockResolvedValue(mockCurso);

      const result = await controller.update(CURSO_ID, {} as any);
      expect(result).toEqual({ success: true, message: 'Course updated successfully', data: mockCurso });
    });

    it('should throw NotFoundException when curso does not exist', async () => {
      mockCursosService.update.mockResolvedValue(null);
      await expect(controller.update(NOT_FOUND_ID, {} as any))
        .rejects.toThrow(NotFoundException);
    });

    it('should call service.update with the correct id and dto', async () => {
      const dto = { nombre: 'Nuevo' } as any;
      mockCursosService.update.mockResolvedValue({ _id: CURSO_ID });
      await controller.update(CURSO_ID, dto);
      expect(mockCursosService.update).toHaveBeenCalledWith(CURSO_ID, dto);
    });

  });

  // ─────────────────────────────────────────────────────────────
  describe('remove()', () => {

    it('should return SuccessResponseDto with deleted curso', async () => {
      const mockCurso = { _id: CURSO_ID, nombre: 'NestJS Avanzado' };
      mockCursosService.remove.mockResolvedValue(mockCurso);

      const result = await controller.remove(CURSO_ID);
      expect(result).toEqual({ success: true, message: 'Course deleted successfully', data: mockCurso });
    });

    it('should throw NotFoundException when curso does not exist', async () => {
      mockCursosService.remove.mockResolvedValue(null);
      await expect(controller.remove(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

    it('should call service.remove with the correct id', async () => {
      mockCursosService.remove.mockResolvedValue({ _id: CURSO_ID });
      await controller.remove(CURSO_ID);
      expect(mockCursosService.remove).toHaveBeenCalledWith(CURSO_ID);
    });

  });

});
```
